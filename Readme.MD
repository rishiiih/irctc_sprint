# ShowTime Ticketing System Architecture

## Full System Diagram

Mobile App / Browser
        │
        │ HTTPS Request
        ▼
┌─────────────────────────────┐
│ CloudFront CDN              │
│ - Static assets             │
│ - Event pages cache         │
│ TTL: 1 hour                 │
└─────────────────────────────┘
        │
        │ Cache Miss / API Call
        ▼
┌─────────────────────────────┐
│ Application Load Balancer   │
│ - SSL Termination           │
│ - Health Check: 10 sec      │
│ - Rate Limit: 200 req/min   │
└─────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────┐
│ Node.js API Cluster (4–20 instances)       │
│ Auto-scale when CPU > 70%                  │
│                                            │
│ Responsibilities:                          │
│ • Authentication                           │
│ • Booking APIs                             │
│ • Redis Lock Acquisition                   │
│ • Cache Reads                              │
│ • Publish Payment Events                   │
└────────────────────────────────────────────┘
        │
        ├─────────────────────────────┐
        │                             │
        ▼                             ▼

┌─────────────────┐        ┌──────────────────┐
│ Redis Cluster   │        │ PostgreSQL       │
│ (3 Nodes)       │        │ Primary          │
├─────────────────┤        ├──────────────────┤
│ Cache Keys      │        │ Write Traffic    │
│                 │        │                  │
│ event:{id}      │        │ Create Booking   │
│ availability:*  │        │ Update Seats     │
│ seatmap:*       │        │ Payment Updates  │
│                 │        │                  │
│ Distributed     │        │                  │
│ Locks           │        │                  │
│ seat_lock:*     │        │                  │
└─────────────────┘        └────────┬─────────┘
                                    │
                                    │ Replication
                                    ▼

                    ┌─────────────────────────┐
                    │ Read Replica 1          │
                    │ Event Details           │
                    │ User Booking History    │
                    └─────────────────────────┘

                    ┌─────────────────────────┐
                    │ Read Replica 2          │
                    │ Seat Map Reads          │
                    │ Availability Queries    │
                    └─────────────────────────┘

        │
        │ Publish Booking Event
        ▼

┌─────────────────────────────┐
│ SQS Payment Queue           │
├─────────────────────────────┤
│ Visibility Timeout: 20 sec  │
│ Max Receive Count: 3        │
│ DLQ Enabled                 │
└─────────────────────────────┘
        │
        ▼

┌─────────────────────────────┐
│ ECS Payment Workers         │
├─────────────────────────────┤
│ 1. Read SQS Message         │
│ 2. Call Razorpay            │
│ 3. Update DB                │
│ 4. Publish Notification     │
│ 5. Delete SQS Message       │
└─────────────────────────────┘
        │
        ▼

┌─────────────────────────────┐
│ AWS SNS                     │
└─────────────────────────────┘
        │
        ├──────────────┐
        ▼              ▼

┌─────────────┐  ┌─────────────┐
│ SES Email   │  │ SMS Gateway │
│ Booking     │  │ Ticket SMS  │
│ Confirmation│  │ Confirmation│
└─────────────┘  └─────────────┘
