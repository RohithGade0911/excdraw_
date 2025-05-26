# Talent Workflow: Consultant/Expert Onboarding Process

## Workflow Overview
This workflow automates the entire talent journey from application submission to project matching and delivery tracking.

## Workflow Structure

### 1. **Trigger: Application Submission**
- **Node Type:** `Webhook` or `Typeform Trigger`
- **Purpose:** Capture talent applications
- **Configuration:**
  ```json
  {
    "httpMethod": "POST",
    "path": "/talent-application",
    "responseMode": "responseNode"
  }
  ```

### 2. **Application Data Processing**
- **Node Type:** `Set`
- **Purpose:** Structure and clean application data
- **Fields to Set:**
  - `talentId` (generate UUID)
  - `fullName`
  - `email`
  - `linkedinProfile`
  - `portfolio`
  - `skills` (array)
  - `experience`
  - `hourlyRate`
  - `availability`
  - `preferredProjectTypes`
  - `applicationDate`
  - `status` (set to "Pending Review")

### 3. **Store Application**
- **Node Type:** `Airtable`
- **Purpose:** Store in talent database
- **Table:** "Talent_Applications"
- **Operation:** Create Record

### 4. **LinkedIn Profile Verification**
- **Node Type:** `HTTP Request` or `LinkedIn API`
- **Purpose:** Verify LinkedIn profile exists and extract data
- **Configuration:**
  ```json
  {
    "method": "GET",
    "url": "https://api.linkedin.com/v2/people/{{linkedinProfile}}",
    "authentication": "oAuth2",
    "headers": {
      "Authorization": "Bearer {{$credentials.LinkedIn.oauthTokenData.access_token}}"
    }
  }
  ```

### 5. **Skills Verification Agent**
- **Node Type:** `AI Agent`
- **Purpose:** Analyze portfolio and verify claimed skills
- **Configuration:**
  ```json
  {
    "model": "gpt-4",
    "prompt": "Analyze the following portfolio and experience description: {{portfolio}} and {{experience}}. Verify if the claimed skills {{skills}} are supported by the evidence. Rate each skill on a scale of 1-10 for credibility and provide recommendations for verification.",
    "output_format": "structured_json"
  }
  ```

### 6. **Automatic Qualification Check**
- **Node Type:** `Code` (JavaScript)
- **Purpose:** Initial automated screening
- **Logic:**
  ```javascript
  // Check minimum requirements
  const hasLinkedIn = items[0].json.linkedinProfile && items[0].json.linkedinProfile !== '';
  const hasPortfolio = items[0].json.portfolio && items[0].json.portfolio !== '';
  const hasRelevantExp = items[0].json.experience && items[0].json.experience.length > 100;
  const skillsVerified = items[0].json.skillsVerificationScore >= 6;
  
  const qualificationScore = 
    (hasLinkedIn ? 25 : 0) +
    (hasPortfolio ? 25 : 0) +
    (hasRelevantExp ? 25 : 0) +
    (skillsVerified ? 25 : 0);
  
  return [{
    json: {
      ...items[0].json,
      qualificationScore: qualificationScore,
      autoQualified: qualificationScore >= 75,
      needsManualReview: qualificationScore < 75 && qualificationScore >= 50,
      autoReject: qualificationScore < 50
    }
  }];
  ```

### 7. **Route Based on Qualification**
- **Node Type:** `Switch`
- **Purpose:** Route applications based on qualification score
- **Rules:**
  - `autoQualified === true` → Continue to background check
  - `needsManualReview === true` → Send to manual review queue
  - `autoReject === true` → Send rejection email

### 8. **Background Check Integration**
- **Node Type:** `HTTP Request`
- **Purpose:** Initiate background verification (for auto-qualified)
- **Service:** Integration with background check service
- **Data Sent:** Name, email, previous employment details

### 9. **Manual Review Queue**
- **Node Type:** `Slack` notification + `Airtable` update
- **Purpose:** Flag for manual review
- **Slack Message:**
  ```
  🔍 New Application Needs Review
  
  Candidate: {{fullName}}
  Skills: {{skills}}
  Experience: {{experience}}
  Qualification Score: {{qualificationScore}}/100
  
  LinkedIn: {{linkedinProfile}}
  Portfolio: {{portfolio}}
  
  Review Application: [Airtable Link]
  ```

### 10. **Update Application Status**
- **Node Type:** `Airtable`
- **Purpose:** Update status based on routing
- **Status Options:**
  - "Background Check in Progress"
  - "Manual Review Required"
  - "Rejected - Does Not Meet Requirements"

### 11. **Applicant Acknowledgment Email**
- **Node Type:** `Gmail`
- **Purpose:** Confirm application received
- **Template:**
  ```html
  Subject: Application Received - Welcome to Outsized Talent Network
  
  Dear {{fullName}},
  
  Thank you for your interest in joining the Outsized talent network!
  
  We've received your application and here's what happens next:
  
  ✅ Application received and logged
  📋 Initial qualification review (24-48 hours)
  🔍 Background and skill verification
  📞 Interview scheduling (if qualified)
  🎉 Welcome to the network!
  
  Application ID: {{talentId}}
  
  You can track your application status here: [Portal Link]
  
  We'll keep you updated throughout the process.
  
  Best regards,
  Outsized Talent Team
  ```

### 12. **Wait for Background Check Results**
- **Node Type:** `Webhook`
- **Purpose:** Receive background check completion
- **Path:** `/background-check-complete/{{talentId}}`

### 13. **Background Check Evaluation**
- **Node Type:** `Switch`
- **Purpose:** Evaluate background check results
- **Rules:**
  - `backgroundCheck.status === "Clear"` → Continue to interview scheduling
  - `backgroundCheck.status === "Issues"` → Manual review
  - `backgroundCheck.status === "Failed"` → Rejection with feedback

### 14. **Interview Scheduling**
- **Node Type:** `Calendly` API
- **Purpose:** Automatically schedule vetting interview
- **Configuration:**
  - Create unique calendar link
  - Set available time slots
  - Include application context
  - Send calendar invite

### 15. **Interview Preparation**
- **Node Type:** `Set` + `Gmail`
- **Purpose:** Prepare interview materials and send prep email
- **Preparation Email:**
  ```html
  Subject: Interview Scheduled - Outsized Talent Network
  
  Dear {{fullName}},
  
  Great news! You've been selected for an interview to join our talent network.
  
  Interview Details:
  📅 Date: {{interviewDate}}
  ⏰ Time: {{interviewTime}}
  🔗 Meeting Link: {{meetingLink}}
  👤 Interviewer: {{interviewerName}}
  
  What to Expect:
  • Portfolio review (15 mins)
  • Experience discussion (15 mins)  
  • Platform overview (10 mins)
  • Q&A (10 mins)
  
  Please prepare:
  ✓ 2-3 best project examples
  ✓ Questions about our platform
  ✓ Your availability for projects
  
  Looking forward to meeting you!
  
  Best regards,
  Outsized Team
  ```

### 16. **Interview Reminder System**
- **Node Type:** `Schedule Trigger` (Multiple)
- **Purpose:** Send reminders before interview
- **Schedule:** 24 hours and 2 hours before interview

### 17. **Post-Interview Processing**
- **Node Type:** `Webhook`
- **Purpose:** Capture interview results and decision
- **Path:** `/interview-complete/{{talentId}}`
- **Data:** Interview score, decision, feedback, interviewer notes

### 18. **Decision Processing**
- **Node Type:** `Switch`
- **Purpose:** Route based on interview outcome
- **Rules:**
  - `decision === "Approved"` → Welcome and onboarding
  - `decision === "Conditional"` → Request additional information
  - `decision === "Rejected"` → Send rejection with feedback

### 19. **Talent Onboarding (Approved Path)**
- **Node Type:** `Sequence` of multiple nodes
  
#### 19a. **Welcome Email**
- **Node Type:** `Gmail`
- **Template:**
  ```html
  Subject: Welcome to Outsized Talent Network! 🎉
  
  Dear {{fullName}},
  
  Congratulations! You've been accepted into the Outsized talent network.
  
  What's Next:
  1. ✅ Complete your profile setup
  2. 📋 Review and sign our talent agreement
  3. 🏦 Set up payment information
  4. 📚 Access our resource portal
  5. 🚀 Start receiving project opportunities!
  
  Your Talent Portal: {{portalLink}}
  Talent Agreement: {{contractLink}}
  
  Welcome to the team!
  
  Best regards,
  Outsized Team
  ```

#### 19b. **Profile Setup**
- **Node Type:** `Airtable` update
- **Purpose:** Move to active talent database
- **Status:** "Active - Onboarding"

#### 19c. **Contract Generation**
- **Node Type:** `PandaDoc`
- **Purpose:** Generate talent agreement
- **Template:** Standard freelancer/consultant agreement

#### 19d. **Payment Setup**
- **Node Type:** `Stripe` or payment processor
- **Purpose:** Create talent payment profile

### 20. **Skills Matching Setup**
- **Node Type:** `Code` (JavaScript)
- **Purpose:** Create talent matching profile
- **Process:**
  ```javascript
  // Create searchable skill tags
  const skillTags = items[0].json.skills.map(skill => ({
    skill: skill.toLowerCase(),
    level: skill.verificationScore,
    category: categorizeSkill(skill)
  }));
  
  // Set availability preferences
  const availability = {
    hoursPerWeek: items[0].json.availability,
    projectTypes: items[0].json.preferredProjectTypes,
    hourlyRate: items[0].json.hourlyRate,
    startDate: items[0].json.earliestStartDate
  };
  
  return [{
    json: {
      talentId: items[0].json.talentId,
      skillTags: skillTags,
      availability: availability,
      status: 'Available'
    }
  }];
  ```

### 21. **Talent Portal Access**
- **Node Type:** `HTTP Request`
- **Purpose:** Create talent portal account
- **Includes:**
  - Dashboard access
  - Project opportunity notifications
  - Profile management
  - Earnings tracking

### 22. **Welcome to Network Notification**
- **Node Type:** `Slack` (Internal)
- **Purpose:** Notify internal team of new talent
- **Message:**
  ```
  🎉 New Talent Added to Network!
  
  Name: {{fullName}}
  Skills: {{skills}}
  Rate: ${{hourlyRate}}/hour
  Availability: {{availability}} hours/week
  
  View Profile: [Talent Portal Link]
  ```

---

## **Project Matching & Assignment Sub-Workflow**

### 23. **New Project Trigger**
- **Node Type:** `Webhook`
- **Purpose:** Triggered when new client project needs talent matching
- **Path:** `/match-talent/{{projectId}}`

### 24. **Talent Matching Agent**
- **Node Type:** `AI Agent`
- **Purpose:** Advanced talent-project matching
- **Configuration:**
  ```json
  {
    "model": "gpt-4",
    "prompt": "Find the best talent matches for this project. Project: {{projectDescription}}, Skills: {{requiredSkills}}, Budget: {{budget}}, Timeline: {{timeline}}. Consider: skill relevance, availability, rate fit, past performance, and client preferences. Rank top 5 candidates with match scores.",
    "context": "talent_database",
    "output_format": "ranked_list"
  }
  ```

### 25. **Availability Check**
- **Node Type:** `Airtable` query
- **Purpose:** Verify matched talent availability
- **Filter:** Status = "Available" AND availability_hours >= project_requirements

### 26. **Talent Notification (Proactive Matching)**
- **Node Type:** `Gmail` (Multiple recipients)
- **Purpose:** Notify matched talent of opportunity
- **Template:**
  ```html
  Subject: New Project Opportunity - {{projectTitle}}
  
  Hi {{talentName}},
  
  We have a new project opportunity that matches your skills!
  
  Project Overview:
  • Title: {{projectTitle}}
  • Duration: {{timeline}}
  • Rate: ${{budgetRange}}/hour
  • Skills: {{requiredSkills}}
  • Start Date: {{startDate}}
  
  You've been shortlisted based on:
  ✓ {{matchingSkills}} skill match
  ✓ Budget alignment
  ✓ Availability fit
  ✓ {{relevantExperience}} experience
  
  Interested? Reply by {{deadlineDate}}
  
  View Full Details: [Project Portal Link]
  Express Interest: [Quick Response Link]
  
  Best regards,
  Outsized Team
  ```

### 27. **Interest Collection**
- **Node Type:** `Webhook`
- **Purpose:** Collect talent interest responses
- **Path:** `/talent-interest/{{projectId}}/{{talentId}}`

### 28. **Shortlist Compilation**
- **Node Type:** `Code` + `Google Docs`
- **Purpose:** Compile final shortlist for client
- **Process:** Aggregate interested talent, rank by match score, create client presentation

---

## **Project Delivery Tracking Sub-Workflow**

### 29. **Project Assignment Trigger**
- **Node Type:** `Webhook`
- **Purpose:** Triggered when talent is selected for project
- **Path:** `/talent-assigned/{{projectId}}/{{talentId}}`

### 30. **Project Workspace Setup**
- **Node Type:** `Asana` or `Monday.com`
- **Purpose:** Create project tracking workspace
- **Include:** Talent and client as collaborators

### 31. **Progress Tracking Setup**
- **Node Type:** `Schedule Trigger` (Weekly)
- **Purpose:** Regular project health monitoring
- **Actions:**
  - Check milestone completion
  - Monitor time tracking
  - Flag potential issues
  - Generate progress reports

### 32. **Milestone Notifications**
- **Node Type:** `Asana Webhook`
- **Purpose:** Automated milestone completion notifications
- **Recipients:** Client, talent, internal team

### 33. **Weekly Status Reports**
- **Node Type:** `Schedule Trigger` + `Gmail`
- **Purpose:** Automated weekly progress reports
- **Frequency:** Every Friday
- **Recipients:** Client and internal team
- **Content:**
  - Completed tasks
  - Upcoming milestones
  - Time spent
  - Budget utilization
  - Issues or concerns

### 34. **Project Completion Processing**
- **Node Type:** `Webhook`
- **Purpose:** Handle project completion
- **Actions:**
  - Generate final report
  - Process final payments
  - Collect feedback from both parties
  - Update talent availability status

---

## **Error Handling & Maintenance**

### 35. **Error Handler**
- **Node Type:** `Error Trigger`
- **Purpose:** Handle workflow failures
- **Actions:**
  - Log error details
  - Notify admin team
  - Attempt recovery
  - Update applicant if necessary

### 36. **Data Cleanup**
- **Node Type:** `Schedule Trigger` (Daily)
- **Purpose:** Maintain data hygiene
- **Actions:**
  - Archive completed applications
  - Update stale records  
  - Sync data between systems
  - Generate daily reports

## Workflow Connections

```
Application → Data Processing → Store → LinkedIn Verification → Skills Agent
                                  ↓
Qualification Check → Route → Background Check / Manual Review / Auto-Reject
                      ↓
Interview Scheduling → Preparation → Reminders → Post-Interview
                                                    ↓
Decision Processing → Welcome/Rejection → Onboarding → Skills Setup
                                                         ↓
Talent Portal → Network Notification → Project Matching → Assignment
                                                            ↓
Project Tracking → Progress Monitoring → Completion → Feedback
```

## Key Features
- **Automated Screening:** AI-powered initial qualification assessment
- **Verification Pipeline:** LinkedIn and background check integration
- **Intelligent Matching:** Advanced AI matching for projects
- **Continuous Monitoring:** Ongoing project and talent performance tracking
- **Scalable Processing:** Handles multiple applications simultaneously
- **Quality Control:** Multiple verification checkpoints
- **Professional Communication:** Automated, personalized messaging throughout the journey