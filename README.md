# API

The public facing API can be accessed using this URL below:

```
https://timegr.id/wp-admin/admin-ajax.php
```

### Table of contents

* [Getting Started](#getting-started)
* [Actions](#actions)

# Getting Started

## Authentification

Every request you send is going to need to include a `security` value inside of the primary data body. This is a temporary nonce value with a validity period of 12 hours for your session. If no such value is included, or the included value has expired, then the request will be automatically declined.

You can start a session and get a nonce value using the action [`start_calendar_session`](#start_calendar_session).

## Data Formatting

All date related resources are in accordance with ISO 8601. Years are defined with all four numbers, and months and days start counting at 1. As such, a date string would be represented as `2025-03-30`.

Days of the week in both requests and responses are formatted as strings, represented by the first 3 letters of that day. For example, Wednesday would be represented in data with `wed`.

## Responses

All data responses returned by API calls have a `success` parameter at the highest level, being set to `true` if the request was successfully processed, or `false` if something went wrong.

On both instances, there will also be a `data` field included in this reponse, but the information inside of this object will depend on whether the operation was successful or not.

If the request was successful, the data contained inside of that `data` object will be tailored to the [action](#actions) you requested, as seen in any of the example response objects for each respective action.

If the request failed, there will always be an `error` field inside of the `data` object with a user friendly message that you can display if something goes wrong. If there was an issue with your nonce code for example, a message starting with `Your session could not be validated at the moment.` would be displayed.

# Actions

The actual operation that you're looking to preform should be defined inside of the `action` parameter of the request body you're sending. The publically accessable actions are as follows:

- **GET** [`start_calendar_session`](#start_calendar_session) - Retrieve a nonce value.
- **GET** [`retrieve_all_account_funnels`](#retrieve_all_account_funnels) - Retrieve the public forms that user's can schedule under.
- **GET** [`retrieve_host_calendar_month_availability`](#retrieve_host_calendar_month_availability) - Retrieve the availability for a specific user for a specific month of a year.
- **GET** [`attempt_public_stripe_key_retrieval`](#attempt_public_stripe_key_retrieval) - Retrieve the public Stripe key that can be used to initialize a Stripe payment form.
- **POST** [`schedule_session_spot`](#schedule_session_spot) - Schedule a session.
- **POST** [`send_single_email`](#send_single_email) - Send an email.

## start_calendar_session

![Version](https://img.shields.io/badge/Method-GET-brightgreen)

Retrieve a nonce value. This is going to be your unique session value that you include alongside every future request in the `security` field.

### Required request data

- **string** `action` - The API action.
- **integer** `user_id` - The numerical ID of the user you're doing this for.

#### Example request data object

``` javascript
{
    action: "start_calendar_session"
    user_id: 6
}
```

### Example response

- **string** `security` - Your session's nonce code.

#### Example response data object

``` javascript
{
    "success": true,
    "data": {
        "security": "e098a090ea"
    }
}
```

## retrieve_all_account_funnels

![Version](https://img.shields.io/badge/Method-GET-brightgreen)

Retrieve the public forms that user's can schedule under.

### Required request data

- **string** `security` - Your session nonce code.
- **string** `action` - The API action.
- **integer** `account_id` - Your account ID.
- **boolean** `open_single_result` - When retrieving this data with the intention to select one, which is the majority of cases, set this to `true`.

#### Example request data object

``` javascript
{
    security: "e098a090ea",
    action: "attempt_public_stripe_key_retrieval",
    account_id: 6,
    open_single_result: true
}
```

### Example response

- **array** `funnels` - An array of publically accessable funnels.
    - **string** `formTitle` - The title of the form.
    - **integer** `ID` - The numerical ID of the form.
    - **boolean** `isFormEnabled` - Are schedulers allowed to book under this form?
    - **boolean** `flexibleBooking` - Should schedulers be allowed to book a variable timeslot instead of a fixed one?
    - **boolean** `arePaymentsEnabled` - Should payment information be collected in order to submit the form?
    - **integer|null** `hourlyPrice` - If payments are enabled, this is the hourly price of the session.
    - **integer|null** `bookingPrice` - If payments are enabled, this is the initial booking price of the session, seperate from the price of the hourly.
    - **integer** `minBook` - The minimum duration in minutes a session needs to be.

#### Example response data object

``` javascript
{
    "success": true,
    "data": {
        "funnels": [
            {
                "formTitle": "Studio Time",
                "ID": 4,
                "isFormEnabled": true,
                "flexibleBooking": true,
                "arePaymentsEnabled": true,
                "hourlyPrice": 50,
                "bookingPrice": null,
                "minBook": 120
            },
            // ... other funnels
        ]
    }
}
```

## retrieve_host_calendar_month_availability

![Version](https://img.shields.io/badge/Method-GET-brightgreen)

Retrieve the availability for a specific user for a specific month of a year. This is how a calendar can be generated, and insight into availability can be seen.

### Required request data

- **string** `security` - Your session nonce code.
- **string** `action` - The API action.
- **integer** `user_id` - The numerical ID of the user we're submitting under.
- **integer** `form_id` - The numerical ID of the form we're submitting under.
- **integer** `selected_year` - The timestamp year.
- **integer** `selected_month` - The timestamp month.
- **integer** `user_timezone` - The timezome offset of the user scheduling the form relative to UTC. This can be between -12 to 14.

#### Example request data object

``` javascript
{
    security: "e098a090ea",
    action: "retrieve_host_calendar_month_availability",
    user_id: 6,
    form_id: 4,
    selected_year: 2025,
    selected_month: 4,
    user_timezone: -4
}
```

### Example response

- **boolean** `flexible_sessions` - Should schedulers be allowed to book a variable timeslot instead of a fixed one?
- **array** `calendar` - The full calendar month, with each sub-array representing each week in that month.
    - **integer** `month_index` - The timestamp month this day falls on.
    - **integer** `day`- The timestamp day.
    - **string** `day_of_week` - The day of the week.
    - **boolean** `is_current_month` - Does this day fall of the same month as the requested month?
    - **string** `date` - The date string.
- **integer** `month_index` - The requested timestamp month.
- **string** `month` - The requested human readable month.
- **integer** `year` - The requested timestamp year.
- **integer** `host_timezone` - The timezome offset of the host relative to UTC.
- **object** `regular_availability` - The regular weekly availability for this user, for this month. Not considering current bookings or time off.
    - **string** `sun`:`sat` - The availability of a specific day of the week.
        - **boolean** `off_day` - Is this specific day unavailable for scheduling?
        - **integer** `start_time` - The minute timestamp between 0 - 1439 where from this point onwards into the day, a session can be scheduled.
        - **integer** `end_time` - The minute timestamp between 0 - 1439 plus the start time being the latest a session can go on until.
- **object** `form_details` - Additional details about the requested form.
    - **boolean** `flexibleBooking` - Should schedulers be allowed to book a variable timeslot instead of a fixed one?
    - **integer** `hostAccountID` - The numerical ID of the user we're submitting under.
    - **integer** `ID` - The numerical ID of the form we're submitting under.
    - **string** `form_title` - The name of the form we're submitting under.
    - **string|null** `time_of_charge` - If payments are enabled, this is the means in which the payment will be processed. This can be:
        - `immediately` - Charge the full amount at the time of booking. Both hourly and initial.
        - `session_start` - Charge the initial booking price at the time of scheduling, but make a second payment for the hourly at the start of the session automatically.
        - `session_end` - Charge the initial booking price at the time of scheduling, but make a second payment for the hourly at the end of the session automatically.
        - `manual` - Charge the inital booking price at the time of scheduling, but leave the remaining hourly for the host to charge manually.
    - **integer** `step_interval` - If this is for a flexible session type, this is the amount of minutes that a session duration can be incremented by. If this is for a fixed session type, this is the total duration of a session.
    - **integer|null** `minimum_booking` - The shortest duration of time a session can be in minutes
    - **integer|null** `maximum_booking` - The longest duration of time a session can be in minutes, with respect to the total availability of a day.
    - **integer|null** `minimum_notice` - The required minimum amount of notice a scheduler must give in hours.
    - **integer|null** `maximum_notice` - The cap of how far into the future someone can schedule a session with you in hours.
    - **integer|null** `hourly_rate` - If payments are enabled, this is the hourly price of the session.
    - **integer|null** `initial_fee` - If payments are enabled, this is the initial booking price of the session, seperate from the price of the hourly.
    - **integer|null** `hourly_amount_charged_on_book` - If payments are enabled and the initial fee is based on a factor of the hourly fee, this is how much of the hourly that should be charged at the time of booking. This is included with the hourly.
    - **boolean** `enable_payments` - Should payment information be collected before submitting?
    - **boolean** `is_form_enabled` - Is this form enabled for public scheduling?
- **array** `sessions` - Blocks of time within this month that are already reserved by someone else.
    - **integer** `ID` - The unique ID of the booking.
    - **integer** `formID` - The numerical ID of the form this was booked under.
    - **string** `sessionDate` - The string date of the session.
    - **integer** `schedulerID` - The unique ID of the booking scheduler.
    - **integer** `sessionStartTime` - The minute timestamp between 0 - 1439 where the session block begins.
    - **integer** `sessionEndTime` - The minute timestamp between 0 - 1439 plus the start time at which the session block ends.
- **array** `time_off` - Blocks of time within this month that are reserved as time off by the host, differing from regular availability.
    - **string** `startDate` - The string date at which this time off begins.
    - **string** `endDate` - The string date at which this time off ends.
    - **integer** `startTime` - The specific time on the start date at which this time off begins.
    - **integer** `endTime` - The specific time on the end date at which this time off ends.
    - **boolean** `openUp` - When true, this should be handled as a block of time that is now available, even when it normally wouldn't be. When false, this should be handled as blocking off this time, even if it was previously available.

#### Example response data object

``` javascript
{
    "success": true,
    "data": {
        "flexible_sessions": true,
        "calendar": [
            [
                {
                    "month_index": 3,
                    "day": 30,
                    "day_of_week": "sun",
                    "is_current_month": false,
                    "date": "2025-03-30"
                },
                {
                    "month_index": 3,
                    "day": 31,
                    "day_of_week": "mon",
                    "is_current_month": false,
                    "date": "2025-03-31"
                },
                {
                    "month_index": 4,
                    "day": 1,
                    "day_of_week": "tue",
                    "is_current_month": true,
                    "date": "2025-04-01"
                },
                {
                    "month_index": 4,
                    "day": 2,
                    "day_of_week": "wed",
                    "is_current_month": true,
                    "date": "2025-04-02"
                },
                // ... more days of the week
            ],
            // ... more weeks
        ],
        "month_index": 4,
        "month": "April",
        "year": 2025,
        "host_timezone": 0,
        "regular_availability": {
            "sun": {
                "off_day": false,
                "start_time": 780,
                "end_time": 1830
            },
            "mon": {
                "off_day": false,
                "start_time": 780,
                "end_time": 1830
            },
            "tue": {
                "off_day": false,
                "start_time": 900,
                "end_time": 1830
            },
            // ... more availability
        },
        "form_details": {
            "flexibleBooking": true,
            "hostAccountID": 6,
            "ID": 4,
            "form_title": "Studio Time",
            "time_of_charge": "session_start",
            "step_interval": 30,
            "minimum_booking": 120,
            "maximum_booking": null,
            "minimum_notice": 24,
            "maximum_notice": null,
            "hourly_rate": 50,
            "initial_fee": null,
            "hourly_amount_charged_on_book": 1,
            "enable_payments": true,
            "is_form_enabled": true
        },
        "sessions": [
            {
                'ID': 144,
                'formID': 4,
                'sessionDate': '2025-04-12',
                'schedulerID': 123,
                'sessionStartTime': 780,
                'sessionEndTime': 840
            }
            // ... other scheduled sessions that month
        ],
        "time_off": [
            {
                'startDate': '2025-04-27',
                'endDate': '2025-04-29',
                'startTime': 900,
                'endTime': 1220,
                'openUp': false
            }
            // ... other time off reserved for that month
        ]
    }
}
```

## attempt_public_stripe_key_retrieval

![Version](https://img.shields.io/badge/Method-GET-brightgreen)

Retrieve the public Stripe key that can be used to initialize a Stripe payment form.

### Required request data

- **string** `security` - Your session nonce code.
- **string** `action` - The API action.

#### Example request data object

``` javascript
{
    security: "e098a090ea",
    action: "attempt_public_stripe_key_retrieval"
}
```

### Example response

- **string** `key` - The Stripe API key.

#### Example response data object

``` javascript
{
    "success": true,
    "data": {
        "key": "pk_live_51QKXwiI..."
    }
}
```

## schedule_session_spot

![Version](https://img.shields.io/badge/Method-POST-purple)

Schedule a session.

### Required request data

- **string** `security` - Your session nonce code.
- **string** `action` - The API action.
- **integer** `session_start_time` - The minute timestamp between 0 - 1439 where the session block begins.
- **integer** `session_end_time` - The minute timestamp between 0 - 1439 plus the start time at which the session block ends.
- **integer** `selected_year` - The timestamp year.
- **integer** `selected_month` - The timestamp month.
- **integer** `selected_day` - The timestamp day.
- **string** `scheduler_name` - The scheduler's name. Maximum accepted length is 150 characters.
- **string** `scheduler_email` - The scheduler's email. Maximum accepted length is 254 characters.
- **string** `scheduler_phone` - The scheduler's phone number
- **integer** `timezone_offset` - The timezome offset of the user scheduling the form relative to UTC. This can be between -12 to 14.
- **string** `token` - Optional. If this form handles payments, the payment information token response from the Stripe form should be included here.
- **integer** `form_id` - The numerical ID of the form we're submitting under. This is how pricing is calculated and charged.

#### Example request data object

``` javascript
{
    security: "e098a090ea",
    action: "schedule_session_spot",
    session_start_time: 720,
    session_end_time: 840,
    selected_year: 2025,
    selected_month: 4,
    selected_day: 28,
    scheduler_name: "John Doe",
    scheduler_email: "abc@example.com",
    scheduler_phone: "+0(111) 222-3333",
    timezone_offset: -4,
    token: "tok_CZ9jS9AICeRTF9iSbHr227s",
    form_id: 4
}
```

### Example response

- **string** `message` - A generic success message.
- **object** `session` - Details about the reserved timeslot.
    - **string** `sessionDate` - The string date of the session.
    - **integer** `sessionStartTime` - The starting minute timestamp of the session between 0 - 1439.
    - **integer** `sessionEndTime` - The ending minute timestamp of the session between 0 - 1439 plus the start time.
- **object** `bill` - Information about the bill behind the session.
    - **float** `totalPrice` - The total price of the session, across both the hourly and the booking price.
    - **float** `remainingPrice` - How much of the total bill is still due. There could be an amount remaining if the `time_of_charge` value is scheduled for the future.
- **object** `host` - Details about the session relating to the host.
    - **string** `publicEmail` - The public email for the host.
    - **string** `publicPhone` - The public phone number for the host.
    - **string** `publicLocation` - The location of the session.

#### Example response data object

``` javascript
{
    "success": true,
    "data": {
        'message' => "Your session was successfully scheduled!",
        "session" => {
            'sessionDate': "2025-04-28",
            'sessionStartTime': 720,
            'sessionEndTime': 840
        },
        "bill" => {
            'totalPrice': 50.0,
            'remainingPrice': 0.0
        },
        "host" => {
            'publicEmail': "host@company.com",
            'publicPhone': "+0(111) 222-3333",
            'publicLocation': "235 Company Location Street GA"
        }
    }
}
```

## send_single_email

![Version](https://img.shields.io/badge/Method-POST-purple)

Send an email. The expected content type for the body is HTML.

### Required request data

- **string** `security` - Your session nonce code.
- **string** `action` - The API action.
- **string** `sender` - The email address that should be listed as the sender.
- **string** `recipient` - The email address recieving this email.
- **string** `subject` - The email subject.
- **string** `body` - The email body.

#### Example request data object

``` javascript
{
    security: "e098a090ea",
    action: "send_single_email",
    sender: "host@company.com",
    recipient: "abc@example.com",
    subject: "Session scheduled!",
    body: "Your email body..."
}
```

### Example response

- **string** `message` - A generic success message.

#### Example response data object

``` javascript
{
    "success": true;
    "data": {
        "message": "Email sent successfully!"
    }
}
```
