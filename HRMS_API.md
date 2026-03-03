# HRMS API Documentation

This document provides a comprehensive list of all API endpoints in the Human Resources Management (HRM) application, including their request and response structures.

## Table of Contents
1. [Production Bootstrapping](#production-bootstrapping)
2. [Authentication & Account](#authentication--account)
3. [Employees](#employees)
4. [Attendance](#attendance)
5. [Work Period Management](#work-period-management-admin-only)
6. [Permission Management](#permission-management)
7. [Leave Management](#leave-management)
8. [Compensatory Off](#compensatory-off)
9. [Company Management](#company-management)
10. [Company Holidays](#company-holidays)
11. [Letter Templates](#letter-templates)
12. [Error Handling](#error-handling)

---

## Production Bootstrapping

When moving the application to a production environment, the system provides an automatic bootstrapping mechanism to solve the "first admin" problem.

### 1. Initial Data Seeding
Upon the first startup, even in production, the `DataInitializer` will:
- Create a **Bootstrap Company** ("Whitecrow Technologies") with Code `WCT-001`.
- Create a **System Admin** account.

### 2. Configuration via Environment Variables
The initial admin credentials can be configured using environment variables. If not provided, defaults are used:
- `BOOTSTRAP_ADMIN_EMAIL`: Defaults to `admin@whitecrowtechnologies.com`
- `BOOTSTRAP_ADMIN_PASSWORD`: Defaults to `admin123`

### 3. First Steps in Production
1. Start the application with your production database.
2. Login using the Bootstrap Admin credentials.
3. Use the `PUT /api/companies/{uuid}` endpoint to update the default company name and address to your actual production details.
4. (Optional) Create a new Admin employee with your personal email and delete the bootstrap admin.

---

## Authentication & Account

### Login
*   **Endpoint**: `POST /api/auth/login`
*   **Description**: Authenticates a user and returns a JWT token.
*   **Real Example**:
    ```bash
    curl -X POST http://localhost:8080/api/auth/login \
         -H "Content-Type: application/json" \
         -d '{
           "email": "admin@whitecrowtechnologies.com",
           "password": "admin"
         }'
    ```
*   **Request Body**:
    ```json
    {
      "email": "admin@whitecrowtechnologies.com",
      "password": "admin"
    }
    ```
*   **Response**: `AuthResponse`
    ```json
    {
      "token": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIi... (truncated)",
      "type": "Bearer",
      "id": 1,
      "email": "admin@whitecrowtechnologies.com",
      "name": "Admin",
      "roles": ["ROLE_ADMIN"]
    }
    ```
*   **Notes**: Only employees with `status: "WORKING"` can log in. accounts with `RESIGNED` or `TERMINATED` status are disabled.

### JWT Token Usage & Parsing

After a successful login, the application returns a JWT (JSON Web Token). This token must be used for all subsequent authorized requests.

#### 1. How to use the token
Include the token in the `Authorization` header of your HTTP requests using the `Bearer` scheme.

**Example Request Header**:
```http
Authorization: Bearer <your_token_here>
```

#### 2. How to parse the token (Client Side)
The token is a standard JWT consisting of three parts separated by dots: `Header.Payload.Signature`.

- **Client-side decoding**: You can use a library like `jwt-decode` (JavaScript) or simply base64-decode the **middle part (Payload)** to see the user information encoded inside (like `sub` for email/username).
- **Server-side verification**: The backend uses the `jjwt` library to verify the signature and integrity of the token.

**Payload Example**:
```json
{
  "sub": "admin@whitecrowtechnologies.com",
  "iat": 1707584148,
  "exp": 1707670548
}
```

### Change Password
*   **Endpoint**: `POST /api/auth/change-password`
*   **Description**: Changes the password for the currently logged-in user.
*   **Security**: Requires Authentication (Bearer Token)
*   **Real Example**:
    ```bash
    curl -X POST http://localhost:8080/api/auth/change-password \
         -H "Authorization: Bearer <your_token>" \
         -H "Content-Type: application/json" \
         -d '{ "newPassword": "newSecurePassword123" }'
    ```
*   **Request Body**: `ChangePasswordRequest`
    ```json
    {
      "newPassword": "string"
    }
    ```
*   **Response**: `String` ("Password updated successfully")

---


## Employees

### Create Employee
*   **Endpoint**: `POST /api/employees`
*   **Security**: Requires ADMIN role
*   **Content-Type**: `multipart/form-data`
*   **Request Parts**:
    - `employee`: `EmployeeRequest` (JSON part)
    - `resume`: `MultipartFile` (PDF)
    - `image`: `MultipartFile` (Image)
    - `provisionalCert`: `MultipartFile` (PDF)
    - `degreeCert`: `MultipartFile` (PDF)
    - `aadhar`: `MultipartFile` (PDF)
    - `pan`: `MultipartFile` (PDF)
    - `sslcCert`: `MultipartFile` (PDF)
    - `hscCert`: `MultipartFile` (PDF)
    - `experience`: `MultipartFile` (PDF)

*   **Employee JSON Data (`employee` part)**:
    ```json
    {
      "email": "string",
      "password": "string (min 8 characters, required for create, optional for update)",
      "name": "string (2-100 characters)",
      "gender": "MALE | FEMALE | OTHER",
      "dateOfBirth": "ISO-8601 string (YYYY-MM-DD)",
      "countryCode": "string (e.g., +91, +1, +44) - Must start with + followed by 1-4 digits",
      "mobileNumber": "string (6-12 digits only)",
      "aadharNumber": "string (12 digits)",
      "salary": "double (positive number)",
      "empId": "string (unique)",
      "role": "ADMIN | EMPLOYEE | HR",
      "designation": "string (optional)",
      "managerId": "long (optional)",
      "teamId": "long (optional)",
      "companyId": "UUID",
      "status": "WORKING | RESIGNED | TERMINATED | ON_LEAVE | SUSPENDED (optional, defaults to WORKING)",
      "leavedAt": "ISO-8601 string (YYYY-MM-DD, optional)",
      "dateOfJoining": "ISO-8601 string (YYYY-MM-DD)",
      "noticePeriod": "string (optional)",
      "degree": "string (optional)",
      "specialization": "string (optional)",
      "collegeName": "string (optional)",
      "yearOfPassing": "string (optional)",
      "street": "string (optional)",
      "area": "string (optional)",
      "city": "string (optional)",
      "district": "string (optional)",
      "state": "string (optional)",
      "pincode": "string (optional)",
      "experiences": [
        {
          "companyName": "string",
          "jobTitle": "string",
          "employmentType": "string",
          "location": "string",
          "startDate": "YYYY-MM-DD",
          "endDate": "YYYY-MM-DD (optional)",
          "description": "string (optional)"
        }
      ]
    }
    ```
*   **Real Example**:
    ```bash
    curl -X POST http://localhost:8080/api/employees \
         -H "Authorization: Bearer <admin_token>" \
         -F 'employee={
           "email": "john.doe@example.com",
           "password": "SecurePass123",
           "name": "John Doe",
           "gender": "MALE",
           "dateOfBirth": "1990-05-15",
           "countryCode": "+91",
           "mobileNumber": "9876543210",
           "aadharNumber": "123456789012",
           "salary": 50000.00,
           "empId": "EMP001",
           "role": "EMPLOYEE",
           "designation": "Software Engineer",
           "companyId": "550e8400-e29b-41d4-a716-446655440000",
           "status": "WORKING",
           "dateOfJoining": "2024-01-01",
           "degree": "B.Tech"
         };type=application/json' \
         -F "resume=@/path/to/resume.pdf" \
         -F "image=@/path/to/photo.jpg" \
         -F "aadhar=@/path/to/aadhar.pdf"
    ```
*   **Response**: `EmployeeResponse`
    ```json
    {
      "id": 1,
      "email": "john.doe@example.com",
      "name": "John Doe",
      "role": "EMPLOYEE",
      "designation": "Software Engineer",
      "managerId": null,
      "managerName": null,
      "teamId": null,
      "teamName": null,
      "companyId": "550e8400-e29b-41d4-a716-446655440000",
      "companyName": "Whitecrow Technologies",
      "empId": "EMP001",
      "gender": "MALE",
      "dateOfBirth": "1990-05-15",
      "countryCode": "+91",
      "mobileNumber": "9876543210",
      "aadharNumber": "123456789012",
      "salary": 50000.0,
      "status": "WORKING",
      "dateOfJoining": "2024-01-01",
      "resume": "resumes/uuid_resume.pdf",
      "employeeImage": "images/uuid_photo.jpg",
      "aadharCard": "aadhar/uuid_aadhar.pdf",
      "panCard": "pan/uuid_pan.pdf",
      "sslcCertificate": "certificates/uuid_sslc.pdf",
      "hscCertificate": "certificates/uuid_hsc.pdf",
      "experienceCertificates": "experience/uuid_exp.pdf",
      "experiences": [
        {
          "companyName": "Previous Corp",
          "jobTitle": "Junior Developer",
          "employmentType": "Full-time",
          "location": "Remote",
          "startDate": "2022-01-01",
          "endDate": "2023-12-31",
          "description": "Worked on various projects."
        }
      ]
    }
    ```
*   **Response**: `EmployeeResponse`
    ```json
    {
      "id": "long",
      "email": "string",
      "name": "string",
      "role": "ADMIN | EMPLOYEE | HR",
      "designation": "string",
      "managerId": "long",
      "managerName": "string",
      "teamId": "long",
      "teamName": "string",
      "companyId": "UUID",
      "companyName": "string",
      "empId": "string",
      "gender": "MALE | FEMALE | OTHER",
      "dateOfBirth": "ISO-8601 string (YYYY-MM-DD)",
      "countryCode": "string (e.g., +91)",
      "mobileNumber": "string (6-12 digits)",
      "aadharNumber": "string (12 digits)",
      "salary": "double",
      "status": "WORKING | RESIGNED | TERMINATED | ON_LEAVE | SUSPENDED",
      "experiences": [
        {
          "companyName": "string",
          "jobTitle": "string",
          "employmentType": "string",
          "location": "string",
          "startDate": "YYYY-MM-DD",
          "endDate": "YYYY-MM-DD (optional)",
          "description": "string"
        }
      ],
      "leavedAt": "ISO-8601 string (YYYY-MM-DD) or null",
      "dateOfJoining": "ISO-8601 string (YYYY-MM-DD)",
      "noticePeriod": "string",
      "degree": "string",
      "specialization": "string",
      "collegeName": "string",
      "yearOfPassing": "string",
      "street": "string",
      "area": "string",
      "city": "string",
      "district": "string",
      "state": "string",
      "pincode": "string"
    }
    ```

### Validation Rules for Employee Fields

#### Country Code
- **Pattern**: `^\+[1-9]\d{0,3}$`
- **Required**: Must start with `+`
- **First digit**: Must be 1-9 (no leading zeros)
- **Length**: 2-5 characters total (e.g., `+1`, `+91`, `+971`)
- **Examples**: 
  - ✅ Valid: `+1`, `+91`, `+44`, `+971`, `+1234`
  - ❌ Invalid: `91` (missing +), `+0` (starts with 0), `+12345` (too long)

#### Mobile Number
- **Pattern**: `^[0-9]{6,12}$`
- **Required**: Yes
- **Format**: Digits only, no spaces or special characters
- **Length**: 6-12 digits
- **Examples**:
  - ✅ Valid: `9876543210`, `5551234`, `501234567`
  - ❌ Invalid: `12345` (too short), `1234567890123` (too long), `555-1234` (contains hyphen)

#### Status
- **Values**: `WORKING`, `RESIGNED`, `TERMINATED`, `ON_LEAVE`, `SUSPENDED`
- **Default**: `WORKING` (automatically set if not provided)
- **Required**: Yes
- **Usage**: Set `leavedAt` date when status is `RESIGNED` or `TERMINATED`. Only employees with `WORKING` status can log in.

#### Gender
- **Values**: `MALE`, `FEMALE`, `OTHER`
- **Required**: Yes

### Get All Employees
*   **Endpoint**: `GET /api/employees`
*   **Security**: Requires ADMIN role
*   **Response**: List of `EmployeeResponse`

### Get Employee By ID
*   **Endpoint**: `GET /api/employees/{id}`
*   **Security**: Requires ADMIN role
*   **Response**: `EmployeeResponse`

### Update Employee
*   **Endpoint**: `PUT /api/employees/{id}`
*   **Security**: Requires ADMIN role
*   **Content-Type**: `multipart/form-data`
*   **Request Parts**: Same as Create Employee (all parts optional).
*   **Response**: `EmployeeResponse`

### Delete Employee
*   **Endpoint**: `DELETE /api/employees/{id}`
*   **Security**: Requires ADMIN role
*   **Response**: No Content (204)

---

## Attendance

### Mark Check-In
*   **Endpoint**: `POST /api/attendance/check-in`
*   **Security**: Requires Authentication (Bearer Token)
*   **Description**: Records a new check-in session for the current day.
*   **Response**: `AttendanceResponse`
    ```json
    {
      "id": 1,
      "date": "2026-02-13",
      "status": "PRESENT",
      "totalWorkingHours": 4.5,
      "permissionHours": 0.0,
      "workPeriodName": "Day Shift",
      "sessions": [
        {
          "id": 10,
          "checkInTime": "2026-02-13T09:00:00",
          "checkOutTime": "2026-02-13T13:30:00",
          "durationHours": 4.5
        }
      ]
    }
    ```

### Mark Check-Out
*   **Endpoint**: `POST /api/attendance/check-out`
*   **Security**: Requires Authentication (Bearer Token)
*   **Description**: Closes the most recent open session for the current day.
*   **Response**: `AttendanceResponse`

### Get My Attendance History
*   **Endpoint**: `GET /api/attendance/my-history`
*   **Request Parameters (Optional)**:
    - `year` (Integer) - e.g., 2026
    - `month` (Integer) - e.g., 2
*   **Security**: Requires Authentication (Bearer Token)
*   **Response**: List of `AttendanceResponse`

### Trigger Absence Check (Admin Only)
*   **Endpoint**: `POST /api/attendance/trigger-absence-check`
*   **Description**: Manually triggers the generation of "ABSENT" records for a specific date (defaults to today).
*   **Security**: Requires ADMIN role
*   **Request Parameter**: `date` (ISO-8601 string, Optional)
*   ```bash
    curl -X POST "http://localhost:8080/api/attendance/trigger-absence-check?date=2026-02-12" \
         -H "Authorization: Bearer <admin_token>"
    ```

### Get All Attendance History (Admin Only)
*   **Endpoint**: `GET /api/attendance/all-history`
*   **Description**: Retrieves attendance records for all employees. Supports filtering by date or month/year. The response is grouped by employee.
*   **Security**: Requires ADMIN role
*   **Request Parameters (Optional)**:
    - `date` (String, YYYY-MM-DD): Filter by specific date
    - `year` (Integer): Filter by year
    - `month` (Integer): Filter by month
*   **Response**: List of `EmployeeAttendanceGroup`
    ```json
    [
      {
        "employeeId": 5,
        "employeeName": "Karthik",
        "attendanceRecords": [
          {
            "id": 1,
            "date": "2026-02-13",
            "status": "PRESENT",
            "totalWorkingHours": 4.5,
            "workPeriodName": "Day Shift",
            "sessions": [
              {
                "id": 10,
                "checkInTime": "2026-02-13T09:00:00",
                "checkOutTime": "2026-02-13T13:30:00",
                "durationHours": 4.5
              }
            ]
          }
        ]
      }
    ]
    ```

---

## Work Period Management (Admin Only)

### Create Work Period
*   **Endpoint**: `POST /api/work-periods`
*   **Description**: Defines a new shift configuration for a company.
*   **Security**: Requires ADMIN role
*   **Request Body**: `WorkPeriodRequest`
    ```json
    {
      "name": "Day Shift",
      "startTime": "09:00:00",
      "endTime": "18:00:00",
      "checkInLimit": "10:00:00",
      "companyId": "550e8400-e29b-41d4-a716-446655440000"
    }
    ```
*   **Response**: `WorkPeriodResponse`

### Get Work Periods by Company
*   **Endpoint**: `GET /api/work-periods/company/{companyId}`
*   **Security**: Requires ADMIN role
*   **Response**: List of `WorkPeriodResponse`

---

## Permission Management

### Apply for Permission (Late Check-in)
*   **Endpoint**: `POST /api/permissions/apply`
*   **Description**: Requests approval for late check-in on a specific date. If approved, the shift `checkInLimit` is extended by the requested hours.
*   **Security**: Requires Authentication (Bearer Token)
*   **Request Body**: `PermissionRequestDto`
    ```json
    {
      "date": "2026-02-14",
      "hours": 2.5,
      "reason": "Family emergency"
    }
    ```
*   **Response**: `PermissionRequestResponse`

### Get Pending Permissions (Admin Only)
*   **Endpoint**: `GET /api/permissions/pending`
*   **Security**: Requires ADMIN role
*   **Response**: List of `PermissionRequestResponse`

### Approve/Reject Permission (Admin Only)
*   **Endpoint**: `PUT /api/permissions/approve/{id}` or `/reject/{id}`
*   **Security**: Requires ADMIN role
*   **Response**: `PermissionRequestResponse`
    ```json
    {
      "id": 1,
      "employeeId": 5,
      "employeeName": "Karthik",
      "date": "2026-02-14",
      "hours": 2.5,
      "status": "APPROVED",
      "reason": "Family emergency",
      "createdAt": "2026-02-13T12:00:00"
    }
    ```

---

## Leave Management

### Get Leave Types
*   **Endpoint**: `GET /api/leave/types`
*   **Description**: Retrieves the list of valid leave types.
*   **Security**: None (Public)
*   **Response**: `List<TypeOption>`
    ```json
    [
      {"label": "Paid Leave", "value": "Paid Leave"},
      {"label": "Sick Leave", "value": "Sick Leave"},
      {"label": "Casual Leave", "value": "Casual leave"},
      {"label": "Compensatory Off", "value": "Compensatory Off"}
    ]
    ```

### Apply for Leave
*   **Endpoint**: `POST /api/leave/apply`
*   **Security**: Requires Authentication (Bearer Token)
*   **Real Example**:
    ```bash
    curl -X POST http://localhost:8080/api/leave/apply \
         -H "Authorization: Bearer <your_token>" \
         -H "Content-Type: application/json" \
         -d '{
           "fromDate": "2026-02-15",
           "toDate": "2026-02-16",
           "leaveType": "SICK",
           "description": "Fever and cold"
         }'
    ```
*   **Request Body**: `LeaveRequestRequest`
    ```json
    {
      "fromDate": "2026-02-15",
      "toDate": "2026-02-16",
      "leaveType": "SICK",
      "description": "Fever and cold"
    }
    ```
*   **Response**: `LeaveRequestResponse`
    ```json
    {
      "id": 1,
      "fromDate": "2026-02-15",
      "toDate": "2026-02-16",
      "leaveType": "SICK",
      "description": "Fever and cold",
      "status": "PENDING",
      "employeeId": 1,
      "employeeName": "Admin"
    }
    ```

### Get My Leaves
*   **Endpoint**: `GET /api/leave/my-leaves`
*   **Security**: Requires Authentication (Bearer Token)
*   **Real Example**:
    ```bash
    curl -X GET http://localhost:8080/api/leave/my-leaves \
         -H "Authorization: Bearer <your_token>"
    ```
*   **Response**: List of `LeaveRequestResponse`

### Update Leave Request (Pending Only)
*   **Endpoint**: `PUT /api/leave/update/{id}`
*   **Description**: Updates a leave request if it is still in "PENDING" status and belongs to the user.
*   **Security**: Requires Authentication (Bearer Token)
*   **Real Example**:
    ```bash
    curl -X PUT http://localhost:8080/api/leave/update/1 \
         -H "Authorization: Bearer <your_token>" \
         -H "Content-Type: application/json" \
         -d '{
           "fromDate": "2026-02-16",
           "toDate": "2026-02-18",
           "leaveType": "SICK",
           "description": "Extended sickness"
         }'
    ```
*   **Request Body**: `LeaveRequestRequest`
*   **Response**: `LeaveRequestResponse`

### Approve Leave (Admin Only)
*   **Endpoint**: `PUT /api/leave/approve/{id}`
*   **Security**: Requires ADMIN role
*   **Real Example**:
    ```bash
    curl -X PUT http://localhost:8080/api/leave/approve/1 \
         -H "Authorization: Bearer <admin_token>"
    ```
*   **Response**: `LeaveRequestResponse`

### Reject Leave (Admin Only)
*   **Endpoint**: `PUT /api/leave/reject/{id}`
*   **Security**: Requires ADMIN role
*   **Real Example**:
    ```bash
    curl -X PUT http://localhost:8080/api/leave/reject/1 \
         -H "Authorization: Bearer <admin_token>"
    ```
*   **Response**: `LeaveRequestResponse`

---

## Compensatory Off

Compensatory Off allows employees to earn credits by working on weekends or company holidays, which they can later use to take time off.

### Request Comp Off Credit
*   **Endpoint**: `POST /api/compoff/request`
*   **Description**: Employee requests a comp off credit for working on a weekend or holiday.
*   **Security**: Requires Authentication (Bearer Token)
*   **Real Example**:
    ```bash
    curl -X POST http://localhost:8080/api/compoff/request \
         -H "Authorization: Bearer <your_token>" \
         -H "Content-Type: application/json" \
         -d '{
           "earnedDate": "2026-02-16",
           "reason": "Worked on Sunday for project deadline"
         }'
    ```
*   **Request Body**: `CompOffRequest`
    ```json
    {
      "earnedDate": "2026-02-16",
      "reason": "Worked on Sunday for project deadline"
    }
    ```
    > **Note**: `earnedDate` must be in the **future or present** (not in the past). This represents the date you plan to work or are working on a weekend/holiday.
*   **Response**: `CompOffResponse`
    ```json
    {
      "id": 1,
      "employeeId": 5,
      "employeeName": "Karthik",
      "earnedDate": "2026-02-16",
      "reason": "Worked on Sunday for project deadline",
      "status": "PENDING",
      "usedDate": null,
      "leaveRequestId": null,
      "createdAt": "2026-02-16"
    }
    ```

### Get My Comp Offs
*   **Endpoint**: `GET /api/compoff/my-compoffs`
*   **Description**: Retrieves all comp off records (pending, approved, rejected, used) for the current user.
*   **Security**: Requires Authentication (Bearer Token)
*   **Real Example**:
    ```bash
    curl -X GET http://localhost:8080/api/compoff/my-compoffs \
         -H "Authorization: Bearer <your_token>"
    ```
*   **Response**: List of `CompOffResponse`

### Get Comp Off Balance
*   **Endpoint**: `GET /api/compoff/balance`
*   **Description**: Returns the number of approved comp off credits available to use.
*   **Security**: Requires Authentication (Bearer Token)
*   **Real Example**:
    ```bash
    curl -X GET http://localhost:8080/api/compoff/balance \
         -H "Authorization: Bearer <your_token>"
    ```
*   **Response**: Long (number)
    ```json
    3
    ```
    *(Employee has 3 comp off days available)*

### Get All Comp Off Requests (Admin Only)
*   **Endpoint**: `GET /api/compoff/all`
*   **Description**: Retrieves all comp off requests from all employees.
*   **Security**: Requires ADMIN role
*   **Real Example**:
    ```bash
    curl -X GET http://localhost:8080/api/compoff/all \
         -H "Authorization: Bearer <admin_token>"
    ```
*   **Response**: List of `CompOffResponse`

### Approve Comp Off (Admin Only)
*   **Endpoint**: `PUT /api/compoff/approve/{id}`
*   **Description**: Approves a comp off credit request. This adds the credit to the employee's balance.
*   **Security**: Requires ADMIN role
*   **Real Example**:
    ```bash
    curl -X PUT http://localhost:8080/api/compoff/approve/1 \
         -H "Authorization: Bearer <admin_token>"
    ```
*   **Response**: `CompOffResponse`
    ```json
    {
      "id": 1,
      "employeeId": 5,
      "employeeName": "Karthik",
      "earnedDate": "2026-02-16",
      "reason": "Worked on Sunday for project deadline",
      "status": "APPROVED",
      "usedDate": null,
      "leaveRequestId": null,
      "createdAt": "2026-02-16"
    }
    ```

### Reject Comp Off (Admin Only)
*   **Endpoint**: `PUT /api/compoff/reject/{id}`
*   **Description**: Rejects a comp off credit request.
*   **Security**: Requires ADMIN role
*   **Real Example**:
    ```bash
    curl -X PUT http://localhost:8080/api/compoff/reject/1 \
         -H "Authorization: Bearer <admin_token>"
    ```
*   **Response**: `CompOffResponse`

### Using Comp Off Credits
To use comp off credits, employees apply for leave with type "Compensatory Off" through the regular leave endpoints:

1. **Apply for Leave**: `POST /api/leave/apply` with `"leaveType": "Compensatory Off"`
   - System validates the employee has sufficient comp off balance
   - If insufficient balance, returns error: "Insufficient comp off balance. Available: X, Required: Y"

2. **Admin Approves Leave**: `PUT /api/leave/approve/{id}`
   - System automatically deducts comp off credits (FIFO - oldest credits first)
   - Comp off status changes to "USED" with `usedDate` and `leaveRequestId` populated

---

## Company Management

### Create Company
*   **Endpoint**: `POST /api/companies`
*   **Security**: Requires ADMIN role
*   **Request**: `CompanyRequest`
    ```json
    {
      "companyName": "string",
      "address": "string",
      "phone": "string",
      "email": "string",
      "website": "string",
      "companyCode": "string"
    }
    ```
*   **Response**: `CompanyResponse`
    ```json
    {
      "id": "UUID",
      "companyName": "string",
      "address": "string",
      "phone": "string",
      "email": "string",
      "website": "string",
      "companyCode": "string"
    }
    ```

### Get All Companies
*   **Endpoint**: `GET /api/companies`
*   **Security**: Requires ADMIN role
*   **Response**: List of `CompanyResponse`

### Get Company By ID
*   **Endpoint**: `GET /api/companies/{uuid}`
*   **Security**: Requires ADMIN role
*   **Response**: `CompanyResponse`

### Update Company
*   **Endpoint**: `PUT /api/companies/{uuid}`
*   **Security**: Requires ADMIN role
*   **Request**: `CompanyRequest`
*   **Response**: `CompanyResponse`

### Delete Company
*   **Endpoint**: `DELETE /api/companies/{uuid}`
*   **Security**: Requires ADMIN role
*   **Response**: No Content (204)

---

## Company Holidays

### Get Holiday Types
*   **Endpoint**: `GET /api/holidays/types`
*   **Description**: Retrieves the list of valid holiday types.
*   **Security**: None (Public)
*   **Response**: `List<TypeOption>`
    ```json
    [
      {"label": "Government Holiday", "value": "Government Holiday"},
      {"label": "Festival Holiday", "value": "Festival Holiday"},
      {"label": "Company Holiday", "value": "Company Holiday"}
    ]
    ```

### Add Holiday
*   **Endpoint**: `POST /api/holidays`
*   **Security**: Requires ADMIN role
*   **Request**: `CompanyHolidayRequest`
    ```json
    {
      "date": "ISO-8601 string",
      "name": "string",
      "type": "string",
      "companyId": "UUID"
    }
    ```
*   **Response**: `CompanyHolidayResponse`
    ```json
    {
      "id": "long",
      "date": "ISO-8601 string",
      "name": "string",
      "type": "string",
      "companyId": "UUID"
    }
    ```

### Add Bulk Holidays
*   **Endpoint**: `POST /api/holidays/bulk`
*   **Description**: Marks specific days of the week as holidays for a range of months.
*   **Security**: Requires ADMIN role
*   **Request**: `BulkHolidayRequest`
    ```json
    {
      "companyId": "UUID",
      "year": "int",
      "startMonth": "int",
      "endMonth": "int",
      "daysOfWeek": ["MONDAY", "TUESDAY", "WEDNESDAY", "THURSDAY", "FRIDAY", "SATURDAY", "SUNDAY"],
      "holidayName": "string",
      "holidayType": "string"
    }
    ```
*   **Response**: List of `CompanyHolidayResponse`

### Remove Bulk Holidays
*   **Endpoint**: `DELETE /api/holidays/bulk`
*   **Description**: Removes holidays for specific days of the week across a range of months.
*   **Security**: Requires ADMIN role
*   **Request**: `BulkHolidayDeleteRequest`
    ```json
    {
      "companyId": "550e8400-e29b-41d4-a716-446655440000",
      "year": 2026,
      "startMonth": 1,
      "endMonth": 12,
      "daysOfWeek": ["SATURDAY"] //All Saturday of the Specified Range Will be Deleted
    }
    ```
*   **Response**: `String` ("Bulk holidays removed successfully")

### Get Holidays By Company
*   **Endpoint**: `GET /api/holidays/company/{companyId}`
*   **Security**: Requires Authentication (Bearer Token)
*   **Response**: List of `CompanyHolidayResponse`

### Update Holiday (Admin Only)
*   **Endpoint**: `PUT /api/holidays/{id}`
*   **Security**: Requires ADMIN role
*   **Request**: `CompanyHolidayRequest`
*   **Response**: `CompanyHolidayResponse`

### Delete Holiday (Admin Only)
*   **Endpoint**: `DELETE /api/holidays/{id}`
*   **Security**: Requires ADMIN role
*   **Response**: `String` ("Holiday deleted successfully")

---

---

## Letter Templates

This section manages generic letter templates that can be used for various purposes like appointment letters, relaying policies, etc.

### Create Letter Template (Admin Only)
*   **Endpoint**: `POST /api/letters`
*   **Description**: Creates a new letter template.
*   **Security**: Requires ADMIN role
*   **Request Body**: `LetterRequest`
    ```json
    {
      "title": "string",
      "description": "string",
      "template": "string (HTML/Text content)"
    }
    ```
*   **Response**: `LetterResponse`
    ```json
    {
      "id": 1,
      "title": "Appointment Letter",
      "description": "Standard template for new hires",
      "template": "<h1>Welcome</h1>..."
    }
    ```

### Get All Letter Templates
*   **Endpoint**: `GET /api/letters`
*   **Description**: Retrieves all stored letter templates.
*   **Security**: Requires Authentication (Bearer Token)
*   **Response**: List of `LetterResponse`

### Get Letter Template By ID
*   **Endpoint**: `GET /api/letters/{id}`
*   **Description**: Retrieves a specific letter template by its ID.
*   **Security**: Requires Authentication (Bearer Token)
*   **Response**: `LetterResponse`

### Update Letter Template (Admin Only)
*   **Endpoint**: `PUT /api/letters/{id}`
*   **Description**: Updates an existing letter template.
*   **Security**: Requires ADMIN role
*   **Request Body**: `LetterRequest`
*   **Response**: `LetterResponse`

### Delete Letter Template (Admin Only)
*   **Endpoint**: `DELETE /api/letters/{id}`
*   **Description**: Deletes a letter template.
*   **Security**: Requires ADMIN role
*   **Response**: No Content (204)

### Generate Letter (Admin Only)
*   **Endpoint**: `POST /api/letters/generate`
*   **Description**: Generates a letter by fetching an employee by email and a template by title, then replacing placeholders.
*   **Security**: Requires ADMIN role
*   **Content-Type**: `application/json`
*   **Accept**: `text/html`
*   **Request Body**: `LetterGenerationRequest`
    ```json
    {
      "email": "employee@example.com",
      "title": "Offer Letter"
    }
    ```
*   **Response**: `String` (HTML content with replaced placeholders)

---

## Error Handling

All API errors return a consistent JSON structure to simplify client-side handling.

### Error Response Format
*   **Response Body**: `ErrorResponse`
    ```json
    {
      "message": "string (Details about the error or concatenated validation errors)"
    }
    ```

### Example: Validation Error
When multiple fields fail validation, the messages are concentrated into a single string:
```json
{
  "message": "email: Email should be valid, password: Password must be at least 8 characters"
}
```

### Example: Runtime Error
```json
{
  "message": "Employee not found"
}
```
