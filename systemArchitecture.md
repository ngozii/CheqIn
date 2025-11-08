# System Architecture - CheqIn
---
## Overview
CheqIn is a GPS-based attendance system that automates employee attendance tracking using geofencing technology. The system consists of a mobile app for employees, a web dashboard for admins, and a backend API that handles geofence validation and attendance records.

### Core Technical Goals:
•	Accuracy: Sub-50 meter geofence precision for reliable check-ins
•	Real-time: Live office presence updates within 30 seconds
•	Scalability: Support 500+ employees per organization
•	Privacy: Location tracking only within office boundaries during work hours
•	Reliability: 99.5% uptime with automated failover mechanisms
•	Security: Multi-layer anti-spoofing detection to prevent GPS manipulation

---

## Technology Stack
### Mobile App (Employee)
•	Framework: React Native 0.73+ with TypeScript
•	Geofencing: geo-toolkits + React Native Geolocation Service
•	State Management: React Context API
•	Background Tasks: react-native-background-actions
•	Push Notifications: Firebase Cloud Messaging (FCM)
•	Local Storage: AsyncStorage for offline data
•	Navigation: React Navigation 

Rationale: React Native enables cross-platform development (iOS & Android) from a single codebase, reducing development time by 60%. TypeScript adds type safety for complex geospatial calculations. FCM provides reliable push notifications across both platforms.

### Admin Dashboard (Web)
•	Framework: React 18 + TypeScript
•	Styling: Tailwind CSS
•	Charts/Analytics: Recharts
•	Data Tables: TanStack Table (React Table v8)
•	Map Integration: Leaflet.js + React-Leaflet for boundary visualization
•	State Management: React Context API + React Query for server state
•	Forms: React Hook Form + Zod validation
Rationale: React ecosystem provides rich components for dashboards. Tailwind enables rapid UI development. Recharts offers lightweight, customizable charts. Leaflet is open-source and doesn't require API keys like Google Maps.


### Backend
•	Runtime: Node.js 20
•	Framework: Express.js
•	Language: TypeScript
•	Geofencing Engine: geo-toolkits (https://www.npmjs.com/package/geo-toolkits)
•	Authentication: JWT tokens with bcrypt password hashing
•	Validation: Zod for input validation
•	API Design: RESTful architecture
•	Rate Limiting: express-rate-limit
•	Security: Helmet.js for HTTP headers
Rationale: Full JavaScript stack reduces context switching. Express is lightweight and has excellent middleware ecosystem. geo-toolkits provides production-ready geofencing without building algorithms from scratch. TypeScript catches bugs during development.


### Database
•	Primary: PostgreSQL 15
•	ORM: Prisma
•	Geospatial Data: GeoJSON format stored as JSONB
•	Indexing: B-tree indexes on user_id, date, and timestamps
•	Full-text Search: PostgreSQL native search for employee lookup
Rationale: PostgreSQL's JSONB support efficiently stores GeoJSON boundaries. ACID compliance ensures attendance record integrity (prevents duplicate check-ins). Prisma provides type-safe database queries that match TypeScript types.


### Infrastructure & Services
•	Backend API: Railway (Node.js hosting with auto-deploy)
•	Admin Dashboard: Vercel (React hosting with CDN)
•	Database: Railway PostgreSQL (managed database)
•	File Storage: AWS S3 for exported reports (CSV/PDF)
•	Push Notifications: Firebase Cloud Messaging
•	Email Service: SendGrid (for invitations and alerts)
•	Version Control: GitHub with protected main branch
Rationale: Railway and Vercel offer free tiers for MVP with seamless GitHub integration. AWS S3 provides reliable, cheap storage for reports. Firebase is free for push notifications up to millions of messages.

---

## System Architecture Diagram
<img src="CheqIn System Architecturee.drawio.png">

---


## Component Interaction & Data Flows
### Employee Registration & Onboarding Flow
1. Admin creates employee account via dashboard
   POST /api/users
   { email, name, department_id, role: "employee" }
   ↓
2. Backend generates temporary password
   Stores user in PostgreSQL
   ↓
3. Backend sends invitation email via SendGrid
   "Welcome to CheqIn! Download the app and use code: ABC123"
   ↓
4. Employee downloads app (iOS/Android)
   ↓
5. Employee enters invitation code + creates password
   POST /api/auth/register-employee
   ↓
6. App requests location permissions
   iOS: "Allow While Using App" or "Allow Always"
   Android: "Allow all the time" for background tracking
   ↓
7. App registers device token with Firebase
   POST /api/devices/register
   { user_id, fcm_token, platform: "ios" }
   ↓
8. Employee is ready for automatic check-ins


### Automatic Check-In Flow
1. App monitors location in background (every 30s)
   React Native Geolocation Service with background mode
   ↓
2. When location changes significantly, app sends:
   POST /api/attendance/check-location
   {
     latitude: 6.5244,
     longitude: 3.3792,
     timestamp: "2025-11-08T08:47:23Z",
     accuracy: 12.5, // meters
     speed: 0.0, // meters/second
     ip_address: "197.210.x.x" // auto-included
   }
   ↓
3. Backend performs anti-spoofing checks:
   a) Validate IP address matches Nigerian geography
   b) Check if speed is physically possible (< 100 km/h)
   c) Verify accuracy is reasonable (< 100m)
   d) Check for GPS spoofing app signatures
   ↓
4. Backend loads office geofence from database
   const office = await prisma.office.findFirst({
     where: { id: user.office_id }
   });
   
   const kit = new GeoToolKit();
   await kit.loadFromJSON(office.geofence);
   ↓
5. Backend checks if inside boundary
   const result = kit.contains(latitude, longitude);
   
   if (result.isInside) {
     // Employee is within geofence
   }
   ↓
6a. IF inside geofence AND not checked in today:
       • Create attendance record
         INSERT INTO attendance (user_id, date, check_in_time, 
                                check_in_location, status)
         VALUES (user_id, CURRENT_DATE, NOW(), 
                POINT(lat, lng), 'present')
       
       • Send push notification via Firebase
         "✅ Checked in at 8:47 AM - Have a great day!"
       
       • Return response
         { 
           status: 'checked_in', 
           time: '2025-11-08T08:47:23Z',
           message: 'Welcome to work!' 
         }
   ↓
6b. IF outside geofence AND currently checked in:
       • Start grace period timer (5 minutes)
       • Check if within lunch hours (12:00-13:00)
         - If lunch: Extended grace period (60 minutes)
         - If not lunch: Standard grace (5 minutes)
       
       • After grace period expires:
         UPDATE attendance
         SET check_out_time = NOW(),
             check_out_location = POINT(lat, lng)
         WHERE user_id = ? AND date = CURRENT_DATE
       
       • Calculate hours worked
         hours = check_out_time - check_in_time
       
       • Send push notification
         "👋 Checked out at 5:47 PM - 8h 44m today"
   ↓
6c. IF suspicious activity detected:
       • Log security event
       • Send alert to admin
       • Optionally: Temporarily disable auto check-in


###  Admin Dashboard Real-Time View
1. Admin logs in via web dashboard
   POST /api/auth/login
   { email, password }
   ↓
2. Backend validates credentials
   Compare bcrypt hash
   Generate JWT token (24h expiry)
   ↓
3. Dashboard loads today's attendance data
   GET /api/attendance/today
   ↓
4. Backend queries PostgreSQL with optimized query:
   SELECT 
     u.id, u.full_name, u.department_id,
     a.check_in_time, a.check_out_time, a.status,
     CASE 
       WHEN a.check_in_time IS NOT NULL 
            AND a.check_out_time IS NULL 
       THEN 'onsite'
       ELSE 'offsite'
     END as presence
   FROM users u
   LEFT JOIN attendance a ON u.id = a.user_id 
     AND a.date = CURRENT_DATE
   WHERE u.role = 'employee'
   ORDER BY a.check_in_time DESC NULLS LAST
   ↓
5. Backend returns aggregated data:
   {
     summary: {
       total_staff: 150,
       present: 127,
       absent: 23,
       late: 8, // checked in after 9 AM
       onsite_now: 105 // currently in office
     },
     staff_list: [
       {
         id: "uuid",
         name: "John Doe",
         department: "Marketing",
         status: "present",
         presence: "onsite", // 🟢 green indicator
         check_in_time: "08:47:23",
         hours_today: "3h 15m"
       },
       // ... more employees
     ]
   }
   ↓
6. Dashboard renders real-time indicators:
   •  Green dot = Onsite (checked in, not checked out)
   •  Gray dot = Offsite (checked out or not arrived)
   •  Red dot = Absent (expected but no check-in)
   ↓
7. Dashboard polls every 30 seconds for updates
   OR uses WebSocket for truly real-time updates (future)



### Geofence Creation Flow (Admin)
1. Admin opens dashboard → "Settings" → "Office Location"
   ↓
2. Dashboard displays interactive Leaflet map
   Centered on current office location or default
   ↓
3. Admin clicks "Edit Geofence" button
   ↓
4. Map enters drawing mode
   Admin can:
   • Draw polygon around office building
   • Drag corners to adjust boundary
   • Set radius (50m - 500m circular geofence)
   ↓
5. Map generates GeoJSON automatically:
   {
     "type": "Feature",
     "geometry": {
       "type": "Polygon",
       "coordinates": [[
         [3.3792, 6.5244],  // lng, lat (Victoria Island)
         [3.3802, 6.5244],
         [3.3802, 6.5234],
         [3.3792, 6.5234],
         [3.3792, 6.5244]
       ]]
     },
     "properties": {
       "office_name": "Lagos HQ",
       "address": "123 Victoria Island",
       "radius_meters": 100
     }
   }
   ↓
6. Admin clicks "Save Geofence"
   PUT /api/offices/:id
   { geofence: { ...GeoJSON... } }
   ↓
7. Backend validates GeoJSON structure
   • Check coordinates are valid (lat: -90 to 90, lng: -180 to 180)
   • Ensure polygon is closed (first = last coordinate)
   • Verify minimum 4 coordinates
   ↓
8. Backend saves to PostgreSQL
   UPDATE offices
   SET geofence = $1::jsonb,
       updated_at = NOW()
   WHERE id = $2
   ↓
9. Backend invalidates cache (if using Redis)
   Backend reloads geofence into geo-toolkits
   ↓
10. All employee apps automatically use new boundary
    on next location check



### Export Attendance Report Flow
1. Admin clicks "Export" button on dashboard
   Selects filters:
   • Date range: Nov 1-8, 2025
   • Department: Marketing (optional)
   • Format: CSV or PDF
   ↓
2. Dashboard sends request:
   GET /api/reports/export?
       start=2025-11-01&
       end=2025-11-08&
       department=marketing&
       format=csv
   ↓
3. Backend queries attendance records:
   SELECT 
     u.full_name, u.email, d.name as department,
     a.date, a.check_in_time, a.check_out_time,
     a.status,
     EXTRACT(EPOCH FROM (a.check_out_time - a.check_in_time))/3600 
       as hours_worked
   FROM attendance a
   JOIN users u ON a.user_id = u.id
   JOIN departments d ON u.department_id = d.id
   WHERE a.date BETWEEN $1 AND $2
     AND d.name = $3 (if filter applied)
   ORDER BY a.date DESC, u.full_name
   ↓
4. Backend generates CSV file:
   Name,Email,Department,Date,Check In,Check Out,Hours,Status
   John Doe,john@company.com,Marketing,2025-11-08,08:47,17:31,8.73,present
   Jane Smith,jane@company.com,Marketing,2025-11-08,09:02,17:45,8.72,late
   ...
   ↓
5. Backend uploads to AWS S3:
   const s3Key = `reports/${orgId}/${timestamp}_attendance.csv`;
   await s3.upload({
     Bucket: 'cheqin-reports',
     Key: s3Key,
     Body: csvBuffer,
     ContentType: 'text/csv'
   });
   ↓
6. Backend generates signed URL (1-hour expiry):
   const url = s3.getSignedUrl('getObject', {
     Bucket: 'cheqin-reports',
     Key: s3Key,
     Expires: 3600 // 1 hour
   });
   ↓
7. Backend returns URL to frontend:
   { 
     download_url: "https://s3.amazonaws.com/...",
     filename: "attendance_2025-11-01_to_2025-11-08.csv",
     expires_at: "2025-11-08T10:47:23Z"
   }
   ↓
8. Dashboard auto-downloads file using the URL
   User's browser saves file to Downloads folder



### Anti-Spoofing Detection Flow
When employee submits location:
├─ Check 1: IP Address Validation
│  ├─ Extract IP from request
│  ├─ Lookup geolocation of IP (MaxMind GeoIP2)
│  └─ If IP country ≠ Nigeria → FLAG
│
├─ Check 2: Movement Speed Analysis
│  ├─ Get last known location & timestamp
│  ├─ Calculate distance using Haversine formula
│  ├─ Calculate speed = distance / time
│  └─ If speed > 100 km/h → FLAG (impossible commute)
│
├─ Check 3: GPS Accuracy Check
│  ├─ Check reported accuracy value
│  └─ If accuracy > 100m → LOW CONFIDENCE
│
├─ Check 4: Pattern Recognition
│  ├─ Check if location jumps erratically
│  ├─ Example: Home → Office → Home in 2 minutes
│  └─ If pattern is suspicious → FLAG
│
└─ If 2+ flags triggered:
   ├─ Block check-in attempt
   ├─ Log security event
   ├─ Send alert to admin: "Suspicious GPS activity from John Doe"
   └─ Return error: "Unable to verify location. Please try again."
________________________________________
________________________________________
}
## GPS Accuracy Considerations

### GPS Accuracy Factors:
•	Open sky (ideal): ±5-10 meters
•	Urban areas: ±20-50 meters (buildings block signals)
•	Indoor/underground: Very poor (50-100+ meters)
•	Weather conditions: Rain/clouds can degrade accuracy

### Geofence Design Best Practices:
1.	Minimum Geofence Radius: 100 meters
o	Accounts for GPS drift
o	Prevents false check-outs from signal fluctuation
o	Balance between security and usability

2.	Building Perimeter + Buffer:
o	Draw geofence around building outline
o	Add 50-100m buffer zone
o	Test with multiple devices before going live

3.	Grace Period Implementation:
o	5-minute standard grace period
o	60-minute lunch grace period
o	Prevents check-out during temporary signal loss

________________________________________

## Trade-offs & Technical Decisions

### Why React Native (Not Native iOS/Android)?
✅ Pros:
•	60% faster development (one codebase for both platforms)
•	Shared business logic reduces bugs
•	Easier to maintain with small team
•	Large community and mature ecosystem
•	Can share TypeScript types with backend
❌ Cons:
•	Slightly worse performance than pure native
•	Background location tracking more complex
•	Limited access to latest platform APIs
•	Larger app bundle size
Decision: React Native is perfect for MVP. The productivity gains outweigh performance concerns. We can always rewrite critical parts in native code later if needed (using native modules).

---

### Why geo-toolkits (Not Google Maps Geofencing API)?
✅ Pros:
•	Free and open source - no API costs
•	No quota limits - unlimited geofence checks
•	Works offline - doesn't need internet after geofence is loaded
•	Full control - can customize algorithms
•	Privacy-friendly - no data sent to Google
❌ Cons:
•	Less battle-tested than Google's solution
•	We maintain the logic ourselves
•	No built-in monitoring/analytics
Decision: geo-toolkits is sufficient for MVP and saves money. Google Geofencing API costs $5-17 per 1,000 requests, which becomes expensive at scale:
•	500 employees × 2 checks/day × 30 days = 30,000 requests/month
•	Cost: $150-510/month just for geofencing
•	vs. geo-toolkits: $0/month
We can always switch to Google later if needed.

---

### Why PostgreSQL (Not MongoDB)?
✅ Pros:
•	Attendance data is highly relational (users ↔ departments ↔ offices)
•	ACID compliance prevents duplicate check-ins
•	JSONB support for flexible GeoJSON storage
•	Powerful querying with JOINs
•	Mature ecosystem and tooling
•	Better for analytics and reporting
❌ Cons:
•	Slightly more complex schema design
•	Migrations can be tricky
Decision: Attendance records have clear relationships. SQL is the right tool for structured, transactional data. MongoDB would be appropriate if we had unstructured event logs, but we don't.

---

### Why JWT (Not Session Cookies)?
✅ Pros:
•	Stateless - no server-side session storage needed
•	Scalable - works well with multiple backend instances
•	Mobile-friendly - easy to use with React Native
•	Cross-domain - works across different subdomains
❌ Cons:
•	Can't invalidate tokens immediately (until expiry)
•	Slightly larger payload than session ID
•	Need to handle token refresh logic
Decision: JWTs are ideal for mobile apps and reduce backend complexity. We mitigate the invalidation issue with short expiry times (24h) and refresh tokens.

---

### Why Not Bluetooth for MVP?
✅ Bluetooth Pros:
•	More accurate (can't be spoofed from home)
•	Works for employees without smartphones
•	Can detect exact entrance used
❌ Bluetooth Cons:
•	Requires hardware installation (Raspberry Pi + BLE readers)
•	Upfront cost: ~$100-200 per entrance
•	Maintenance required (replace batteries, fix broken readers)
•	Slower to market (need to ship hardware to each client)
Decision: GPS-first approach lets us:
1.	Validate market demand quickly (no hardware delays)
2.	Serve 100 companies without physical installation
3.	Add Bluetooth as premium feature later for high-security clients
This is a classic "software beats hardware" strategy for MVP speed.

---

## Future Enhancements
### Phase 1: MVP 
•   [x] GPS geofencing with geo-toolkits
•   [x] Mobile app (iOS/Android)
•	[x] Admin dashboard
•	[x] Real-time presence indicator
•	[x] CSV export
•	[x] Push notifications
•	[x] Anti-spoofing detection

### Phase 2: Core Features 
•	[ ] Multi-location support: Employees work at different branch offices
•	[ ] Department policies: Different geofence rules per department
•	[ ] Email digests: Daily/weekly attendance summaries to managers
•	[ ] Shift scheduling: Support for 24/7 operations
•	[ ] Overtime tracking: Automatic calculation of extra hours

### Phase 3: Advanced Features 
•	[ ] Bluetooth readers: Hybrid GPS + Bluetooth for high security and inclusivity
•	[ ] Biometric verification: Optional face recognition for check-in
•	[ ] Advanced analytics: 
o	Punctuality trends
o	Department comparisons
o	Predictive insights (who's likely to be late)
•	[ ] Mobile widgets: Quick status view without opening app
•	[ ] Smartwatch support: Apple Watch and Wear OS apps

### Phase 4: Enterprise 
•	[ ] White-label solution: Rebrand for enterprise clients
•	[ ] Visitor management: Check-in system for guests
•	[ ] Geofence-based tasks: Assign tasks when employees enter certain zones
•	[ ] Compliance reporting: GDPR, SOC 2, ISO 27001 audit trails
•	[ ] Custom integrations: Zapier, Microsoft Teams, Slack notifications
•	[ ] Multi-tenant architecture: Support multiple organizations in single instance
•	[ ] API marketplace: Public API for third-party developers
•	[ ] Machine learning: Anomaly detection, fraud prediction

---


## Alerting (Better Uptime / PagerDuty)
### Critical Alerts (Immediate):
•	API downtime > 2 minutes
•	Database connection failures
•	Error rate > 5% for 5 minutes
•	Failed check-ins > 20% for 10 minutes
### Warning Alerts (Review within 1 hour):
•	Response time > 500ms for 10 minutes
•	GPS spoofing attempts > 50/hour
•	Failed push notifications > 10%
### Info Alerts (Daily digest):
•	Number of check-ins today
•	New users registered
•	Reports generated

---  

## Maintenance Tasks
### Daily:
•	[ ] Review error logs (Sentry dashboard)
•	[ ] Check API response times (Railway metrics)
•	[ ] Monitor database size and growth
### Weekly:
•	[ ] Review security events (GPS spoofing attempts)
•	[ ] Analyze slow queries and optimize
•	[ ] Check for outdated dependencies
•	[ ] Review user feedback and bug reports
### Monthly:
•	[ ] Database backup verification (test restore)
•	[ ] Update dependencies (security patches)
•	[ ] Performance testing (load tests)
•	[ ] Review and rotate API keys/secrets
### Quarterly:
•	[ ] Security audit (penetration testing)
•	[ ] Database cleanup (archive old records)
•	[ ] Review and update documentation
•	[ ] Capacity planning (forecast growth)

---


## Deployment Architecture
Deployment Checklist
### Pre-Deployment:
•	[ ] All tests passing (unit, integration, e2e)
•	[ ] Database migrations reviewed
•	[ ] Environment variables set
•	[ ] Security audit completed
•	[ ] Performance testing done
•	[ ] Backup current production database

### Deployment Steps:
1.	Merge to main branch
2.	git checkout main
3.	git merge develop
4.	git push origin main
5.	Database migration (if needed)
6.	railway run npx prisma migrate deploy
7.	Deploy backend (automatic via Railway)
o	Railway detects push to main
o	Runs build process
o	Health check endpoint: GET /health
o	Zero-downtime deployment
8.	Deploy frontend (automatic via Vercel)
o	Vercel detects push to main
o	Builds React app
o	Deploys to global CDN
9.	Mobile app release
o	iOS: Upload to App Store Connect
o	Android: Upload to Google Play Console
o	Phased rollout (10% → 50% → 100%)

### Post-Deployment:
•	[ ] Verify all API endpoints working
•	[ ] Test mobile app check-in flow
•	[ ] Check Sentry for new errors
•	[ ] Monitor response times for 1 hour
•	[ ] Send announcement to users (if major update)

### Rollback Plan
If deployment fails:
# Revert backend to previous Railway deployment
railway rollback

# Revert frontend to previous Vercel deployment
vercel rollback

# Revert database migration (if needed)
railway run npx prisma migrate reset

---

##  Risk Assessment & Mitigation
### Technical Risks
Risk 1: GPS Inaccuracy
•	Impact: False check-ins, user frustration
•	Likelihood: Medium
•	Mitigation: 
o	Implement 100m minimum geofence radius
o	5-minute grace period for fluctuations
o	Anti-spoofing detection
o	Allow manual override by admin

Risk 2: Battery Drain
•	Impact: Users disable location → system breaks
•	Likelihood: Medium
•	Mitigation: 
o	Use "significant location change" API
o	Check location every 30s (not continuously)
o	Optimize background tasks
o	Educate users about battery impact

Risk 3: Database Downtime
•	Impact: No check-ins recorded
•	Likelihood: Low
•	Mitigation: 
o	Daily automated backups
o	Railway's 99.95% uptime SLA
o	Offline mode (queue failed requests)
o	Real-time monitoring + alerts

Risk 4: API Rate Limiting
•	Impact: Users can't check in during peak hours
•	Likelihood: Low
•	Mitigation: 
o	Implement request queuing
o	Scale backend horizontally
o	Use connection pooling
o	Cache frequent queries

### Business Risks
Risk 5: Privacy Concerns
•	Impact: Users refuse to adopt
•	Likelihood: Medium
•	Mitigation: 
o	Transparent privacy policy
o	Track location ONLY at work
o	Users can view their own data
o	GDPR compliance
o	Optional "privacy mode" (manual check-in)

Risk 6: Competitor Copies Idea
•	Impact: Loss of market share
•	Likelihood: High
•	Mitigation: 
o	Speed to market (MVP in 3 months)
o	Build strong brand and UX
o	Focus on customer relationships
o	Add unique features (hybrid GPS+Bluetooth)
o	File patents if applicable

Risk 7: Regulatory Changes
•	Impact: Need to rebuild features
•	Likelihood: Low
•	Mitigation: 
o	Follow GDPR best practices
o	Work with legal counsel
o	Monitor employment law changes
o	Build flexible, modular architecture

---

## Success Metrics & KPIs
Product Metrics
User Adoption:
•	[ ] 10 paying organizations by Month 3
•	[ ] 50 organizations by Month 6
•	[ ] 2,500 total employees tracked
•	[ ] 80%+ daily active usage rate

Technical Performance:
•	[ ] 99.5% uptime
•	[ ] < 200ms API response time (p95)
•	[ ] < 0.5% error rate
•	[ ] 95%+ check-in success rate

User Satisfaction:
•	[ ] 4.5+ star rating on app stores
•	[ ] Net Promoter Score (NPS) > 50
•	[ ] < 5% churn rate
•	[ ] 80% would recommend to others

---
---


## Conclusion

CheqIn's architecture is designed for reliability, scalability, and developer productivity. By leveraging proven technologies (React Native, Node.js, PostgreSQL) and specialized libraries (geo-toolkits), we can deliver a robust attendance system that eliminates time theft while respecting employee privacy.
### Core Architectural Strengths
1.	Automated & Seamless: Zero user input required for check-in/out
2.	Secure by Design: Multi-layer anti-spoofing, encrypted data, GDPR compliant
3.	Scalable Foundation: Starts with 500 employees, scales to 10,000+
4.	Privacy-First: Location tracking only at work, full transparency
5.	Cost-Effective: $7-12/month for MVP, grows with revenue
6.	Fast Time-to-Market: GPS-first approach ships faster than hardware

### Technical Highlights
•	geo-toolkits: Production-ready geofencing without building from scratch
•	React Native: Cross-platform mobile app from single codebase
•	PostgreSQL: ACID compliance prevents duplicate check-ins
•	TypeScript: Type safety across frontend, backend, and mobile
•	Background Tracking: Automatic location monitoring with battery optimization
•	Real-time Dashboard: Live presence indicators for admins

---


Document Metadata
Version: 1.0
Last Updated: November 8, 2025
Author: Mercy Ohabuike
Status: Ready for Development
Review Cycle: Monthly during development, quarterly post-launch

---

Appendix: Additional Resources
Useful Links
•	geo-toolkits Documentation: https://www.npmjs.com/package/geo-toolkits
•	GeoJSON Editor: https://geojson.io
•	React Native Docs: https://reactnative.dev
•	Prisma Docs: https://www.prisma.io/docs
•	PostgreSQL PostGIS: https://postgis.net (if we need advanced geospatial queries)
•	Firebase Cloud Messaging: https://firebase.google.com/docs/cloud-messaging

