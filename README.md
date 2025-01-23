# **Implementation Plan: Self-Serve Customer Lifecycle**

# **Table of Contents**

1. [Problem Statement](#problem-statement)  
2. [Overall Solution](#overall-solution)  
3. [Implementation Plan](#implementation-plan)  
    - [Step 1.1: Lifecycle Dashboard for Sales Reps (Quick Win)](#step-11-lifecycle-dashboard-for-sales-reps-quick-win)  
    - [Step 1.2: Trial Expiration Alerts (Quick Win)](#step-12-trial-expiration-alerts-quick-win)  
    - [Step 2: Error Tracking and Monitoring (Foundation for Fault Tolerance)](#step-2-error-tracking-and-monitoring-foundation-for-fault-tolerance)  
    - [Step 3: Integration to Receive Trial Data](#step-3-integration-to-receive-trial-data)  
    - [Step 4: Automate Workspace Record Creation](#step-4-automate-workspace-record-creation)  
    - [Step 5: Workspace Linking (Leads and Accounts)](#step-5-workspace-linking-leads-and-accounts)  
    - [Step 6: Data Privacy & Compliance](#step-6-data-privacy-compliance)  
    - [Step 7: Testing and Rollout](#step-7-testing-and-rollout)  
4. [How Requirements and Events Are Addressed](#how-requirements-and-events-are-addressed)  
    - [Requirements Fulfillment](#requirements-fulfillment)  
    - [Events Coverage](#events-coverage)  
5. [Summary of Action Items](#summary-of-action-items)  
6. [Actors Involved](#actors-involved)  

---

## **Problem Statement**
Sales representatives currently struggle to effectively manage self-serve customers due to:
1. **Incomplete Salesforce Records**: `Leads`, `Workspaces`, and `Accounts` require manual creation and often have missing data
2. **Delayed Status Updates**: Product lifecycle changes (trials, conversions, churn) aren't immediately reflected in Salesforce
3. **Limited Visibility**: Without proper dashboards or alert systems, sales teams can't effectively prioritize trials or respond to churned customers
4. **Current Sales Rep Experience**
![frustrated-rep](images/frustrated-rep.webp)

### **Current Go-To-Market Stack Integration**
- Marketo (Marketing Automation) creates initial leads
- Salesforce (CRM) stores customer records
- Salesforce CPQ handles quotes and orders
- Stripe manages billing
- Snowflake warehouses data

### **Key Assumptions**
1. **Existing Salesforce Configuration**:
   - `Workspace` custom object is already configured
   - User permissions and sharing rules are properly set up
   - Standard objects (`Lead`, `Account`, `Contact`) are configured per requirements

2. **Integration Infrastructure**:
   - Marketo is connected to Salesforce, requiring only process improvements
   - Stripe integration exists with the provisioning system
   - Platform Events feature is enabled in Salesforce
   - API limits and governance limits are sufficient for real-time events
   - Salesforce CPQ is configured to handle self-serve to sales-led conversions

3. **Data Quality**:
   - Email addresses in the provisioning system are valid and standardized
   - Workspace IDs are unique and consistent
   - Trial duration rules are standardized (30 days)

4. **Technical Environment**:
   - Development and testing environments (sandboxes) are available
   - Change management process exists for deployments
   - Monitoring tools are available for system health checks

5. **Business Process**:
   - Sales territory rules and lead assignment logic exist
   - Lead conversion process is defined
   - SLA requirements for data sync are defined

Note: We've based our plan on these assumptions about the current system. If we discover any differences during implementation, we'll work together to adjust our approach accordingly.

### **Event Processing Sequence**
1. **Trial Signup Flow**:
   ```
   Platform Event Received (New Trial)
   └─► Create/Update Workspace
       └─► Find/Create Lead (via Email)
           └─► Find/Link Account (via Domain)
   ```

2. **Trial Conversion Flow**:
   ```
   Platform Event Received (Paid Conversion)
   └─► Update Workspace Status
       └─► Convert Lead to Account/Contact
           └─► Update Workspace Relationships
   ```

3. **Churn Flow**:
   ```
   Platform Event Received (Churn)
   └─► Update Workspace Status
   ```

### **Out-of-Order Handling**
- Remove this entire section since it's now detailed in Step 4

### **Step 3: Integration to Receive Trial Data**

#### **Objective**
Enable real-time reception of customer lifecycle events.

#### **Tasks**
1. **Event Reception Infrastructure**:
   - Configure Platform Events:
     - `SelfServeTrialEvent__e`:
       ```
       Fields:
       - Workspace_ID__c: Unique identifier
       - Email__c: Customer email
       - Event_Type__c: (Trial/Conversion/Churn)
       - Event_Data__c: JSON payload including:
         - For Trial: Trial expiration date
         - For Conversion: Subscription details, plan type, MRR
         - For Churn: Churn date, reason if available
       - Event_Timestamp__c: When event occurred
       ```
   - Configure Platform Event settings:
     - Set replay ID retention period
     - Configure publish behavior
     - Set delivery SLA alerts
   - Create event storage object for caching:
     - `Event_Cache__c`:
       - `Event_Type__c`: Type of event
       - `Event_Data__c`: JSON of event data
       - `Processing_Status__c`: New/Processing/Completed/Error
       - `Retry_Count__c`: Number of retries
       - `Original_Timestamp__c`: When event occurred
       - `Processing_Notes__c`: Details of processing attempts

2. **Event Processing Framework**:
   - Configure event handlers for each type (Trial/Conversion/Churn)
   - Set up error handling and monitoring
   - Create scheduled job for cached events: (this can be either a batch job or a scheduled flow)

#### **Requirements Met**
- Enables real-time updates for lifecycle events
- Provides foundation for real-time data warehouse updates

#### **Events Covered**
- **New trial provisioned**.
- **Trial converts to paid**.
- **Customer churns**.

---

### **Step 4: Automate Workspace Record Creation**

#### **Tasks**
1. **Trial Event Processing Flow**:
   ```
   Receive Trial Event
   └─► Check for existing Workspace
       ├─► If exists: 
       │   ├─► If Paid: Update trial history
       │   └─► If Trial/Churned: Update status
       └─► If new: Create Workspace
           └─► Trigger Lead/Account linking
   ```

2. **Conversion Event Processing Flow**:
   ```
   Receive Conversion Event
   └─► Check for existing Workspace
       ├─► If exists: Update to Paid
       └─► If new: 
           ├─► Create Workspace (Paid)
           ├─► Cache trial data for history
           └─► Flag for review if no trial data found
   ```

3. **Churn Event Processing Flow**:
   ```
   Receive Churn Event
   └─► Check for existing Workspace
       ├─► If exists: Update to Churned
       └─► If new:
           ├─► Create Workspace (Churned)
           ├─► Cache event
           └─► Flag for review
   ```

4. **Out-of-Order Handling**:
   ```
   For Any Event:
   └─► Check Event_Cache__c for related events
       ├─► If found:
       │   ├─► Process cached events in chronological order
       │   └─► Update all related records
       └─► If none:
           └─► Process current event
           └─► Cache if needed for future reference

   Scheduled Process:
   └─► Query Event_Cache__c for unresolved events
       └─► For each cached event:
           ├─► Attempt reprocessing
           ├─► Update historical data
           └─► Mark as processed or increment retry count
   ```

5. **Error Handling**:
   ```
   For Any Processing Error:
   └─► Log error details
   └─► Cache original event
   └─► Flag for manual review if:
       ├─► Max retries exceeded
       ├─► Critical data missing
       └─► Conflicting data found
   ```

#### **Requirements Met**
- Automates foundational data creation and updates lifecycle stages.

#### **Events Covered**
- **New trial provisioned**.
- **Trial converts to paid**.
- **Customer churns**.

---

### **Step 5: Workspace Linking**

#### **Objective**
Link `Workspace` records to `Leads` and `Accounts` for complete data relationships.

#### **Tasks**
1. **Lead Processing Flow**:
   ```
   Workspace Created/Updated
   └─► Check for existing Lead
       ├─► If exists: Link to Workspace
       └─► If none: Create new Lead
   ```

2. **Account Matching Flow**:
   ```
   After Lead Processing
   └─► Extract email domain
   └─► Search for matching Account
       ├─► If found: Link to Workspace
       └─► If none: Queue for review
   ```

3. **Conversion Handling Flow**:
   ```
   Paid Status Update
   └─► Check Account status
       ├─► If Account exists:
       │   └─► Convert Lead to Contact
       └─► If no Account:
           └─► Convert Lead to Account/Contact
   ```

4. **Marketo Sync Handler**:
   ```
   Lead Created in Marketo
   └─► Search for Workspace
       ├─► If found: Link records
       └─► If none: Cache for later
   ```

#### **Requirements Met**
- Ensures `Workspace` records are fully linked to marketing and sales data.

#### **Events Covered**
- **New trial provisioned**
- **Trial converts to paid**
  - Lead conversion to Account/Contact
  - Existing Account handling

---

### **Step 6: Data Privacy & Compliance**

#### **Objective**
Ensure all automated data handling complies with privacy regulations.

#### **Tasks**
1. Implement data retention policies
2. Configure field-level security
3. Set up audit trails for automated record creation
4. Document data handling procedures
5. Create process for handling customer data requests

#### **Requirements Met**
- Ensures compliance with data privacy regulations
- Protects customer data

#### **Events Covered**
- All customer data handling events

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
Key deliverables:
- Error tracking system with monitoring dashboard
- Sales lifecycle visibility dashboard
- Real-time data integration via Platform Events
- Automated record creation and linking
- Trial expiration notification system
- Phased testing and deployment plan

---

## **Actors Involved**

- **Customer**: End user who initiates trials and uses the product
- **Provisioning System**: Backend service sending lifecycle events
- **Salesforce Admin**: System architect managing integration and monitoring
- **Sales Representative**: Dashboard user acting on customer insights
- **Marketo**: Marketing automation system creating initial leads

