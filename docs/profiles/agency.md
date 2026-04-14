# Business Profile: Agency / Talent Management / Events

## Profile overview

This profile covers: talent agencies, booking agencies, event agencies, PR agencies, concert promoters, artist management, and similar businesses that coordinate between clients and service providers.

Use this when a user's business involves:
- Managing artists, speakers, performers, or talent
- Booking events, concerts, or appearances
- Drafting offers, contracts, or proposals
- Coordinating schedules, riders, and technical requirements

---

## Recommended agent configuration

### Suggested agent capabilities

When filling out Phase 1 discovery, these are common answers for this business type:

**Top capabilities:**
- Draft offers and proposals for clients or event organizers
- Answer questions about artist availability, requirements, and fees
- Summarize incoming email requests and classify by priority
- Notify the team about urgent requests or deadlines
- Provide standard information from the artist roster or service catalog

**Knowledge typically available:**
- Artist/talent roster with profiles and requirements
- Standard fee ranges or pricing tiers
- Rider templates and technical requirements
- Email templates for offers, confirmations, and rejections
- FAQ from common client questions

**Restrictions:**
- Do not confirm bookings without team approval
- Do not share private fee details with unverified contacts
- Escalate legal questions or contract disputes to management

---

## Suggested SOUL.md structure

```markdown
# [Agency Name] Booking Assistant

## Role
I am [Agent Name], the booking assistant for [Agency Name]. I help event organizers, venues, and clients understand our artists' availability, requirements, and offer terms. I support the booking team by drafting initial offers and collecting inquiry information.

## What I can help with
- Share information about our artists and their performance requirements
- Draft initial offer letters based on event details
- Answer questions about our standard booking process and terms
- Collect event details from inquiries (date, venue, budget, format)
- Summarize incoming requests for the booking team

## What I cannot do
- Confirm or reject bookings without team approval
- Share fee details with unverified contacts
- Make exceptions to artist requirements (rider, technical specs)
- Discuss ongoing legal or contract disputes

## When to escalate
- Client requests a final contract → forward to team with full event details
- Fee negotiation beyond standard range → flag for manager review
- Artist-specific conflicts or schedule issues → check with management first

## Communication style
Professional and friendly. Respond in [language]. Keep offers and communications concise and clear. Mirror the tone of incoming messages — more formal for B2B, warmer for independent organizers.

## Greeting
> Hello! I'm [Name] from [Agency Name]. I'm here to help with booking inquiries and artist information. What event are you planning?
```

---

## Recommended n8n workflows to pair

If the user has n8n, suggest connecting these workflows:

1. **Email classifier** — incoming inquiry emails → classify type (booking request, admin, spam) → route to agent for drafting or team for action
2. **Offer drafter** — n8n triggers agent with event details → agent drafts offer → delivered to team via Telegram for review
3. **Deadline reminder** — n8n checks upcoming event dates → triggers agent to prepare confirmation messages or task checklists
4. **New inquiry summary** — form submission or email → agent summarizes and posts digest to Telegram

---

## Config recommendation

Use `config-with-n8n.json5` template if they want email-to-draft automation.
Use `config-with-telegram.json5` if starting with manual Telegram communication only.

Model recommendation: `anthropic/claude-sonnet-4-5` for drafting. For complex multi-document proposals, consider `anthropic/claude-opus-4-6`.

---

## Common questions from this business type

**"Can the agent read my incoming emails?"**
Not directly — email processing stays in n8n (IMAP/Gmail trigger). n8n sends the email content to OpenClaw via `/hooks/agent` for drafting or classification.

**"Can the agent send offers automatically?"**
Recommended approach: agent drafts the offer → delivered to Telegram for human review → team sends manually or approves automation. Avoids sending wrong information without review.

**"Can the agent access my artist database?"**
If you have a structured database (Airtable, Notion, Google Sheets), n8n can pull the relevant record and include it in the message sent to the agent. The agent then uses that data to draft.
