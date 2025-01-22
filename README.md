# **Salesforce Implementation Plan: Self-Serve Customer Lifecycle**

---

## **Problem Statement**
Sales reps face inefficiencies in prospecting and upgrading self-serve customers because:
1. **Self-Serve Customers Are Not Fully Represented in Salesforce**: Records such as `Leads`, `Workspaces`, and `Accounts` are inconsistently created, requiring manual intervention.
2. **Real-Time Updates Are Missing**: Product lifecycle events (e.g., trial, paid, churned) are not reflected in Salesforce in real-time.
3. **Sales Reps Lack Actionable Insights**: Without dashboards or alerts, reps struggle to prioritize trials and churned customers.

---

## **Overall Solution**
Develop an automated, fault-tolerant solution that:
1. Automatically creates and links `Workspace`, `Lead`, and `Account` records.
2. Updates Salesforce in real-time with product lifecycle events via Platform Events or REST API.
3. Provides actionable dashboards and notifications for sales reps and admins to act on customer data.

---

## **Implementation Plan**

### **Step 1: Lifecycle Dashboard for Sales Reps (Quick Win)**

#### **Objective**
Provide immediate visibility into customer lifecycle stages (`Trial`, `Paid`, `Churned`) for sales reps.

#### **Tasks**
1. Use existing `Workspace` data to build a **Lifecycle Dashboard**:
   - Display `Trial`, `Paid`, and `Churned` customers.
   - Include filters for trial expiration.
2. Deliver this dashboard to sales reps for immediate use.

#### **Requirements Met**
- Enables sales reps to take action on customer lifecycle data without waiting for full automation.

#### **Events Covered**
- None directly, but builds on existing `Workspace` data.

---

### **Step 2: Error Tracking and Monitoring (Foundation for Fault Tolerance)**

#### **Objective**
Establish the foundation for error tracking and monitoring, providing visibility into the integration’s health from the start.

#### **Tasks**
1. **Create the `Error Log` Custom Object**:
   - Fields:
     - `Error Name` (Text, 255)
     - `Error Type` (Picklist: `Data Issue`, `Integration Failure`, etc.)
     - `Error Details` (Long Text Area)
     - `Source Process` (Picklist: `Workspace Creation`, `Lead Link`, etc.)
     - `Associated Record` (Lookup: Any Object)
     - `Status` (Picklist: `Open`, `Resolved`)
     - `Retry Count` (Number)
     - `Date Occurred` (Date/Time)
2. **Build an Error Tracking Dashboard**:
   - Visualize unresolved errors by type, source, and frequency.
3. **Prepare Future Automation**:
   - Define error logging best practices for all Flows, Triggers, and integrations.
4. **Integrate Temporary Error Logging**:
   - Log issues from manual processes into the `Error Log`.

#### **Requirements Met**
- Establishes fault tolerance and monitoring early.
- Provides visibility into errors during development and rollout.

#### **Events Covered**
- Logs errors for all events:
  - Data ingestion (Platform Events or REST API).
  - Manual and automated processes as they are developed.

---

### **Step 3: Integration to Receive Trial Data**

#### **Objective**
Enable Salesforce to receive lifecycle events (Trial, Paid, Churned) in real-time.

#### **Tasks**
1. Define the integration method:
   - **Preferred**: Platform Events (`SelfServeTrialEvent__e`).
   - **Alternative**: REST API endpoint.
2. Create the `SelfServeTrialEvent__e` Platform Event:
   - Fields:
     - `Workspace_ID__c`, `Email__c`, `Self_Serve_Status__c`, etc.
3. Work with the provisioning system to send events in real-time.
4. Test event ingestion using sample data.

#### **Requirements Met**
- Enables real-time updates for lifecycle events.

#### **Events Covered**
- **New trial provisioned**.
- **Trial converts to paid**.
- **Customer churns**.

---

### **Step 4: Trial Expiration Alerts (Quick Win)**

#### **Objective**
Notify sales reps of trials nearing expiration for proactive follow-up.

#### **Tasks**
1. Build a **scheduled Flow or report** to identify trials expiring in the next 7 days.
2. Configure notifications (email or Salesforce Custom Notifications) for sales reps.
3. Optionally include metrics like trial usage or start date.

#### **Requirements Met**
- Increases conversion opportunities by enabling proactive outreach.

#### **Events Covered**
- **New trial provisioned** (with trial expiration date).

---

### **Step 5: Automate Workspace Record Creation**

#### **Objective**
Automatically create and maintain `Workspace` records in Salesforce.

#### **Tasks**
1. Build a **Platform Event-triggered Flow**:
   - Create `Workspace` records for new trials.
   - Update lifecycle stage for existing `Workspace` records.
2. Populate fields like `Workspace Name`, `Self Serve Status`, and `Email`.
3. Log errors (e.g., duplicate or missing records) to the `Error Log`.

#### **Requirements Met**
- Automates foundational data creation and updates lifecycle stages.

#### **Events Covered**
- **New trial provisioned**.
- **Trial converts to paid**.
- **Customer churns**.

---

### **Step 6: Workspace Linking (Leads and Accounts)**

#### **Objective**
Link `Workspace` records to `Leads` and `Accounts` for complete data relationships.

#### **Tasks**
1. Build a **record-triggered Flow**:
   - Query for a matching `Lead` using `Workspace.Email`.
   - If no match, create a new `Lead` and link it to the `Workspace`.
2. Build another Flow:
   - Extract the domain from `Workspace.Email` and query for a matching `Account`.
   - Link `Workspace` to `Account` or log unmatched records in the `Error Log`.

#### **Requirements Met**
- Ensures `Workspace` records are fully linked to marketing and sales data.

#### **Events Covered**
- **New trial provisioned**.
- **Trial converts to paid**.

---

### **Step 7: Testing and Rollout**

#### **Objective**
Validate the system and deploy incrementally.

#### **Tasks**
1. Conduct end-to-end testing in a sandbox environment:
   - Validate Flows, Platform Events, and dashboards.
2. Roll out in phases:
   - **Phase 1**: Pilot with a small sales team.
   - **Phase 2**: Incremental expansion to all teams.
3. Monitor dashboards during rollout to track adoption and errors.

#### **Requirements Met**
- Ensures reliability and smooth deployment.

#### **Events Covered**
- Validates all events and processes.

---

## **How Requirements and Events Are Addressed**

### **Requirements Fulfillment**

| **Requirement**                                   | **Plan Step Addressing It**                                       |
|---------------------------------------------------|-------------------------------------------------------------------|
| Self-serve customers represented in Salesforce    | Steps 3, 5, and 6.                                               |
| Near-real-time updates                            | Step 3 (Platform Events).                                        |
| Sales program enablement                          | Steps 1, 4, and 6 (dashboards, alerts, data relationships).       |
| Fault tolerance and error monitoring              | Step 2 (Error Log and tracking).                                 |

### **Events Coverage**

| **Product Event**          | **Salesforce Action**                                      | **Plan Step Addressing It**                  |
|----------------------------|----------------------------------------------------------|---------------------------------------------|
| New self-serve trial        | Create `Workspace`, link to `Lead`, notify sales reps.    | Steps 3, 4, 5, and 6.                       |
| Trial converts to paid      | Update `Workspace`, link to `Account` and `Contact`.     | Steps 3, 5, and 6.                          |
| Customer churns             | Update `Workspace.Self Serve Status = Churned`.          | Steps 3 and 5.                              |

---

## **Summary of Action Items**

- Build `Error Log` object and dashboards.
- Set up Lifecycle Dashboard for sales reps.
- Implement real-time data ingestion via Platform Events.
- Automate `Workspace` creation and linking.
- Notify sales reps of expiring trials.
- Test thoroughly and roll out in phases.

---

## **Actors Involved**

- **Customer**: Signs up for trials and interacts with the product.
- **Provisioning System**: Sends product lifecycle events to Salesforce.
- **Salesforce Admin**: Oversees integration, error monitoring, and dashboard creation.
- **Sales Rep**: Uses dashboards and notifications to act on customer data.
- **Marketo**: Creates `Leads` for trial signups and syncs them with Salesforce.

