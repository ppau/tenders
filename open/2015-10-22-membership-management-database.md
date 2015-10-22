# Introduction

Below is the a tender for the creation of a utility for Pirate Party Australia.

The process for applying is quite simple.

1. Write up your proposal, including total cost and delivery schedule.
2. Send it to tenders@pirateparty.org.au.

If you have any questions about any content below, you may email the same address above.

**Applications close 23 November, 2015.**

# Tender: Membership Management Database

## Versions

* **1.0 (2015-10-22)**: initial proposal (current)

## Overview

The Pirate Party needs a membership database to manage the day-to-day operations of the Party.

This database and web application will be used for at least the following purposes:

* Creating and updating membership by a prospective or current member
* Handling invoicing and the surrounding data requirements of invoicing for renewals and donations
* Generating the necessary documentation to remain registered as a federal political party
* Handling different membership types (full, supporter, etc)
* Sending membership announcements to specific membership types
* Automatically send renewal reminders and notifications
* Fine-grained access control to different sections of membership data for different roles such as secretary, treasurer, auditor, or state-specific roles.

The current system has no interface, and only implements a subset of these items. A full rewrite will be necessary for a coherent and accessible membership database going forward.

The source code will be open sourced under a GPLv3 license.

## Proposed Implementation

To consider an approach to implementing the requirements of the system, consideration of the most common use cases is appropriate, which is that of the new member.

### New Member use case

The prospective member's perspective:

1. Prospective member loads the web form.
2. Prospective member completes the form, choosing from a series of membership types and payment options
3. The form is submitted to the server by the prospective user
4. The data is validated for accuracy
5. The member is emailed a confirmation email to determine that the email address is valid.
6. The member accepts, and the now accepted member is provided welcome documentation, and payment invoice if necessary.
7. If necessary, the member pays via the chosen method.

The auditor's perspective:

1. The auditor logs into the web interface for the database.
2. The auditor accesses the audit view and sees all the new members.
3. The auditor checks the data provided against the electoral roll using the AEC website.
4. If necessary, the auditor corrects any obviously incorrect information. The Secretary will be notified of any corrections.
5. The auditor confirms that this information is correct and confirms the member, or refers the membership to the Secretary.
6. The auditor is no longer able to see the member information, and all actions by the auditor have been logged.

The treasurer's perspective:

1. The treasurer logs into the web interface for the database.
2. The treasurer observes all pending payments that require attention from the relevant view.
3. The treasurer cross-references the reference codes (such as for direct deposit) with the relevant accounts.
4. The treasurer updates the status where relevant.
5. The treasurer no longer is able to see the member information, and all actions by the treasurer have been logged.

The secretary's perspective:

1. The secretary logs into the web interface for the database.
2. The secretary checks their notifications view for any actions they are requested to take.
3. If necessary, pending actions are reviewed and actions are taken where necessary.
4. The secretary may also review any actions that have recently occurred in a logging view.
5. As usual, any actions are logged.

----
As we can see from this single use case, the mere creation of a new member creates a significant workload for the Party, and an audit trail is necessary to ensure it all works out.

In practice, a lot of this can be automated. Payments will often occur via digital means and can automatically update the database, limiting the work the treasurer must do.

Auditing is necessary to ensure the membership database remains clean and up to date. Taking an approach of doing this incrementally ensures that AEC audits take no longer than merely exporting a database.

### Proposed Approach

#### Backend

Assuming technology used is **Node.js** for the web interface and **MongoDB** as the database backend. Language should be **TypeScript** in order to implement some semblance of type safety, and allow implementation of actual interfaces for code readability and contract enforcement.

The web interface consists largely of the following concepts:

* Views to a database query
* Fine-tunable permissions per user roles
* Logging, logging, logging

A view in most cases will consist of an access list with roles that grant access to said view, and a JavaScript function (eg a complex query) specifically for determining which results are visible or accessible. It will also of course have a template for displaying the results.

For example, let us assume there are several secretaries with the role `state-secretary`. The query for the membership editor should take into account the secretary's state flag, so for example:

User object:
```javascript
{
  username: "nsw-secretary",
  roles: ["state-secretary"],
  flags: {
    secretaryState: "nsw"
  }
}
```

A slightly pseudo-code query function for the view:

```javascript
let query;

if (hasRole(user, 'secretary')) {
  query = {};
} else if (hasRole(user, 'state-secretary')) {
  query = {
    "details.residential_state": user.flags.secretaryState
  }
} else {
  throw new AccessViolation("Attempted access to restricted view!", user)
}

return yield this.render('membership-editor', {
  members: yield models.Member.find(query).exec();
});
```

A higher level middleware should log the access violation, and should be at a sufficient level that the Secretary is immediately emailed.

Access violations should also be handled at a higher level, but for the sake of defensive coding, it is added here as well.

For simplicity's sake, follow a constraint of not creating sub-permissions within the views themselves, and instead create separate views for such situations (eg an auditing view for the `auditor` role, with a more advanced member editing view for `secretary`). This simplifies the work needed to be done within the templates, with only a little bit of duplication in some cases. It enforces separation of concerns and limits the ability to make mistakes with overcomplicated queries.

#### Logging

**Log everything.** Attach the user to each log item. Log with detail.

Have access violations and other severe log items immediately emailed to the Secretary and potentially a security issue response team if the capacity exists to do so.

### Pretty forms

They get a very pretty form. It should be easy to use and as forgiving as possible.

Preferred technologies would be **Angular.js** or **React.js** (perhaps a combination of the two if rational).

Ideally, the form would be modifiable from the administrative backend without changing HTML, ie data-driven. A TOML-based configuration would be excellent.

Specifically, such a domain-specific language would need to interact with:

- Visibility rules
- Conditional fields
- Membership types

The [Oyster voting system](https://bbqsrc.github.io/oyster) uses TOML for configuring and defining poll information. Data driven, fo sho.

The current membership form could be retrofitted to support such a model (in theory, YMMV).

### Proposed management interface

#### Security

* Log everything.
* Two-factor authentication for everyone, no exceptions.
* Enforce minimum password lengths.
* Enforce changing passwords.
* Assume the frontend is buggy as fuck and everything is always leaking. **LOG EVERYTHING.**
* Assume the server is buggy as fuck and literally built out of Heartbleed. **LOG EVERYTHING EVERYWHERE.**
* No but seriously, log everything. Have monitoring tools that notify everyone when anything looks a bit odd. Two false positives a day is better than one false negative ever.

#### Views

* Neat, purposeful templates with strong permissions enforcement.
* Specific views considered necessary at this point are:
  * User creation and management
  * Log viewer
  * Membership editor
  * Membership list view for auditors, secretaries, and treasurers with relevant columns and actions
  * Mass mailing interface
  * Membership welcome email editor
  * Exporters (CSV, etc as required by AEC)

### Proposed architecture and priorities

The architecture is simply a web service that connects to the relevant APIs, receives and validates forms data, and provides access to the membership database using the prescribed view model with rich queries and permissions enforcement.

The priorities are:

1. Create the relevant models (User, Member, etc) and the authentication, logging and security enforcement procedures prior to starting any other work.
2. User management interface and logging views are an immediate dependency following the completion of the core items. Assume for now the existence only of the `secretary` role.
3. Implement the payment gateway APIs.
4. Implement and integrate a membership form, the membership form editor, membership welcome email editor, the membership editor and list view for the `secretary` role at a minimum.
5. Implement the automatic renewals and renewal reminders subsystems
6. Implement the treasurer's list and related tools for managing payments, and consider the introduction of the `treasurer` role at this point.
7. Add a mass mailing interface for the `secretary` role.
8. Implement auditor's views and relevant tools, and consider the introduction of the `auditor` role at this point.
9. Implement exporters as necessary.
10. Introduce `state-secretary` and other state-specific roles as necessity becomes apparent, and modify queries as necessary.

At this point, all requirements should be met as outlined in the overview.

#### Models

At a minimum, it is expected that there will be the following models:

* **User**: a user of the management interface
* **Member**: a member of the organisation
* **LogItem**: a log record
* **Invoices**: active invoices should be linked to a **Member** as required

#### Views

Views should be implemented on top of **Koa** or a similar HTTP framework. It is unlikely that it will be necessary to add permission queries to the database, and can be coded into the routing of the web app.

If this is deemed necessary at a later point, it can be broken out into a database model.

#### Subsystems

* Renewal reminders: for timed tasks like sending out reminders.
* Payment gateway interfaces: should be able to handle both receiving confirmation from payment gateways, and integrating with invoicing suites such as Xero for automatic reporting. A generic interface should be developed to interact consistently with other models and interfaces, so payments can be triggered easily without API-specific knowledge.
* Membership processor: new members, member payments and information updates should all be handled by the membership form and its interface.
* Views interface: this is largely **Koa** itself, but is listed for clarity. This interview will interact will mostly interact with the membership process, and the database models directly.

At the very least, the membership processor needs to support the following actions:

* New
* Update
* Payment (similar to update, but with a request for payment also)
* Audit (URL endpoints for an email, to click "yes", "update" or "resign")

## Deadlines

At the very least, the core Secretarial infrastructure and renewal infrastructure must be delivered by **15 January, 2016**. All other deliverables must be delivered by **15 March, 2016** at the latest.
