# WordPress Newsletter System with Mailster and Amazon SES

This project documents how I set up a cost-effective, high-deliverability newsletter system for [MamboCity London](https://mambocity.co.uk) using:

- [Mailster](https://mailster.co/) for newsletter creation and subscriber management
- [Amazon SES](https://aws.amazon.com/ses/) for reliable bulk email sending at near-zero cost
- [mailster-amazonses](https://github.com/EverPress/mailster-amazonses) for automated bounce and complaint handling via SNS
- Google Analytics via MonsterInsights for full reader behaviour tracking beyond the email

The solution sends approximately monthly campaigns to ~9,000 opted-in subscribers directly from WordPress, with automated bounce and complaint handling and three independent analytics layers that outperform any commercial email platform.

---

## Architecture

Mailster connects directly to Amazon SES via IAM credentials. Bounce and complaint notifications are routed back from SES through Amazon SNS to Mailster automatically.

> **Mailster (Create) → Amazon SES (Deliver) → SNS (Notify) → Mailster (Suppress)**

**Note:** FluentSMTP was used in the initial configuration as a routing bridge but was subsequently removed. Mailster now connects to SES directly via IAM credentials, which simplified the architecture and improved reliability.

---

## Cost

| Component | Cost |
|---|---|
| Mailster licence | $199 one-off (CodeCanyon) |
| Amazon SES | ~$0.10 per 1,000 emails |
| Cost per send to 9,000 subscribers | ~$0.90 (~£0.72) |
| Annual cost (approx. monthly sends) | ~£9 per year |

### vs Commercial Platforms (9,000 subscribers)

| Platform | Monthly cost | Annual saving vs this solution |
|---|---|---|
| Mailchimp | ~£87–107 | ~£1,071–1,431 |
| Constant Contact Standard | ~£128 | ~£1,527 |
| Constant Contact Premium | ~£220 | ~£2,631 |
| Klaviyo | ~£100–150 | ~£1,191–1,791 |
| **This solution (AWS SES)** | **~£0.72 per send** | **Baseline** |

---

## Tech Stack

- WordPress (self-hosted via SiteGround)
- Mailster (premium plugin, $199 one-off licence)
- mailster-amazonses v3.0.0 (EverPress add-on for SNS bounce/complaint handling)
- Amazon Simple Email Service (SES): eu-west-2 (London)
- Amazon SNS: ses-bounce-v2 and ses-complaint-v2 topics
- Google Workspace (regular inbox email, preserved alongside SES)
- DNS and hosting managed via SiteGround

---

## Step-by-Step Setup

### 1. Domain, Email & DNS Configuration

- **Domain**: `mambocity.co.uk`
- **Registrar & DNS**: SiteGround
- **Email hosting**: Google Workspace
- **SES domain verification**: Completed in AWS console
- **DNS records added on SiteGround**:
  - SPF: `v=spf1 include:_spf.google.com include:amazonses.com ~all`
  - DKIM: 3x CNAME records provided by SES, all verified
  - DMARC: `v=DMARC1; p=none; pct=100; rua=mailto:re+mc5ppzgl1sh@dmarc.postmarkapp.com; sp=none; aspf=r; adkim=r;`
  - MAIL FROM: `mail.mambocity.co.uk` (subdomain, preserves Google MX on root domain)
- Google MX records preserved for regular inbox use alongside SES

### 2. Mailster Configuration & SES Integration

- Mailster premium plugin installed via CodeCanyon
- ~9,000 subscribers uploaded via CSV
- Mailing lists created and configured
- Opt-in forms and profile update pages set up
- Unsubscribe links enabled and tested
- In Mailster delivery settings: Amazon SES selected as delivery method, IAM Access Key and Secret Key entered directly

### 3. AWS SES Configuration

- IAM user created with `AmazonSESFullAccess` policy
- Access Key and Secret Key stored securely in Mailster delivery settings
- Region: `eu-west-2` (London)
- MAIL FROM domain configured: `mail.mambocity.co.uk`
- Production access granted (50,000 emails/day, 14 per second)
- SES Configuration Set created with event destinations for bounce and complaint events
- Test email sent and confirmed, passing SPF and DKIM checks

### 4. Automated Bounce and Complaint Handling

SES bounce and complaint notifications are automatically routed back into Mailster via Amazon SNS, suppressing affected subscribers without manual intervention.

**Components:**

- Two SNS topics: `ses-bounce-v2` and `ses-complaint-v2`
- SES Configuration Set with event destinations pointing to each topic
- mailster-amazonses plugin (v3.0.0) subscribed to both topics via HTTPS endpoint
- SNS HTTPS subscriptions with Raw message delivery disabled

**What happens automatically:**

- **Hard bounce**: subscriber status set to Hardbounced; address suppressed from future sends
- **Spam complaint**: subscriber unsubscribed from all lists

**Root cause resolved:** The mailster-amazonses plugin originally checked only for `notificationType` in the SNS message body, which is the field used by older SES identity-level notifications. SES Configuration Set event destinations send `eventType` instead. The plugin's `handle_sns()` method was updated to check `notificationType` first and fall back to `eventType`:

```php
switch ( isset( $message->notificationType ) ? $message->notificationType : $message->eventType ) {
```

**SES suppression list sync:** All 93 addresses previously on the SES account-level suppression list were exported via AWS CLI and imported into Mailster (73 as Hardbounced, 20 as Unsubscribed) to bring Mailster's records into alignment with SES.

### 5. Integration Testing

- Internal test campaign created and sent via Mailster
- Delivery, formatting and unsubscribe links confirmed working
- Open and click tracking confirmed active in Mailster analytics
- Bounce handling tested using SES simulator address (bounce@simulator.amazonses.com): subscriber automatically set to Hardbounced
- Complaint handling confirmed via activity log: subscriber automatically unsubscribed
- Scaled up to full list send after successful tests

---

## Newsletter Production Workflow

Campaigns use a deliberate two-stage approach that separates content delivery from email delivery.

**Stage 1: Full newsletter as a WordPress page**
The complete newsletter is built as a dedicated page on mambocity.co.uk. Because it is a standard WordPress page, it is automatically tracked by Google Analytics via MonsterInsights, capturing page views, scroll depth, time on page and outbound link clicks. The page remains editable after the email has been sent.

**Stage 2: Teaser email via Mailster**
A shorter teaser campaign is sent to the full subscriber list via SES, containing a brief editorial intro and a link to the full WordPress page. Mailster tracks opens and clicks on the teaser; Google Analytics tracks reader behaviour on the page itself.

---

## Analytics

This system delivers three independent analytics layers that exceed what any commercial platform provides:

| Layer | Tool | Data captured |
|---|---|---|
| 1 | Mailster | Opens, clicks, unsubscribes, bounces (per campaign) |
| 2 | Google Analytics (MonsterInsights) | Page views, scroll depth, time on page, outbound link clicks |
| 3 | Amazon SES console | Delivery rate, bounce rate, complaint rate (infrastructure level) |

**Advantage over commercial platforms:** Mailchimp, Constant Contact and Klaviyo track only what happens inside the email (opens and link clicks). This system additionally captures full reader behaviour on the newsletter page, providing a complete picture from email open through to action taken.

### Performance (as of November 2026)

| Metric | Value |
|---|---|
| Subscribers | ~9,000 |
| Campaigns sent | 27+ |
| Typical delivery rate | 99%+ |
| Open rate | 40–49% (industry average: 20–25%) |
| Bounce handling | Fully automated via SES → SNS → Mailster |
| Complaint handling | Fully automated via SES → SNS → Mailster |

---

## Challenges & Solutions

### MAIL FROM Domain MX Configuration
**Challenge:** The SES MAIL FROM domain requires its own MX record, which initially conflicted with Google Workspace MX records.

**Solution:** Configured the MAIL FROM as a subdomain (`mail.mambocity.co.uk`), keeping Google MX records on the root domain. Both now coexist correctly.

### SES Production Access
**Challenge:** SES starts in sandbox mode, limiting sending to verified addresses only.

**Solution:** Submitted a use-case description explaining the 9,000 opted-in subscriber list, send frequency and compliance practices. Production access granted within 3 days.

### IAM Permissions and DNS Setup
**Challenge:** IAM permission configuration and DNS record propagation required careful sequencing.

**Solution:** Created a dedicated IAM user with `AmazonSESFullAccess` policy. Confirmed DKIM verification in SES console before testing. Waited 24 hours for DNS propagation before retesting.

### SNS Notification Field Mismatch
**Challenge:** The mailster-amazonses plugin silently failed to process bounce and complaint notifications. SES Configuration Set event destinations send `eventType` in the SNS message body, but the plugin only checked for `notificationType` (the older direct SES identity notification format).

**Solution:** Updated `amazonses.class.php` line 640 to check `notificationType` first and fall back to `eventType`:
```php
switch ( isset( $message->notificationType ) ? $message->notificationType : $message->eventType ) {
```
Also confirmed that SNS subscriptions must have Raw message delivery disabled, so the SNS envelope (including the `Type: Notification` wrapper) is preserved for the plugin to parse correctly.

---

## Best Practices Implemented

- SPF, DKIM and DMARC all configured, protecting domain reputation and improving deliverability
- Dedicated MAIL FROM subdomain, separating transactional sending reputation from the main domain
- Dedicated IAM user for SES sending, with tighter least-privilege permissions planned
- Google MX records preserved, so regular inbox email is unaffected by SES configuration
- Unsubscribe links in every campaign, ensuring GDPR and CAN-SPAM compliance
- Automated bounce and complaint suppression via SES → SNS → Mailster, protecting sender reputation
- Hardbounced and unsubscribed addresses retained in Mailster for GDPR audit trail and to prevent re-sending
- CSV subscriber upload with opt-in verification, with no cold or purchased lists
- SES suppression list synced to Mailster on initial setup

---

## Outstanding Issues

- **DMARC policy**: currently `p=none` (monitoring only). To be tightened to `p=quarantine` once deliverability is consistently strong over several months
- **IAM permissions**: IAM user currently uses `AmazonSESFullAccess`. To be replaced with a least-privilege custom policy scoped to sending only

---

## Resources

- [Mailster Plugin](https://codecanyon.net/item/mailster-email-newsletter-plugin-for-wordpress/3078294)
- [mailster-amazonses Add-on](https://mailster.co/add-ons/amazon-ses/)
- [Amazon SES Documentation](https://docs.aws.amazon.com/ses/latest/dg/)
- [Amazon SNS Documentation](https://docs.aws.amazon.com/sns/latest/dg/)
- [SiteGround DNS Management](https://www.siteground.com/kb/how_to_manage_dns_records/)

---

## License

MIT License

---

## Author

**Robert White**, Co-Director, MamboCity London

- [mambocity.co.uk](https://mambocity.co.uk)
- [LinkedIn](https://www.linkedin.com/in/robert-r-white-00252738/)
- [GitHub](https://github.com/Roberto-Blanco)

---

Feel free to fork or adapt this setup for your own WordPress + SES newsletter needs!
