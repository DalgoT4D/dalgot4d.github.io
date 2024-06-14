## Dalgo Webhook configuration

All orchestration flow runs (scheduled or manual) are executed in Prefect by these workers. Dalgo needs to be notified when these flow runs reach a terminal state (success or failure) in order to clear up resources & notify users of failures.

Steps to create a webhook in Prefect:

1. Go to the Prefect UI & head over to Notifications

<img width="1114" alt="image" src="https://github.com/DalgoT4D/dalgot4d.github.io/assets/2160416/84a288c4-cbf3-4c28-ba66-010cc09b1caf">

2. Add a new notification of type Custom Webhook.

3. Set Webhook URL to http://localhost:8002/webhooks/v1/notification/ (assuming the Django server is listening on http://localhost:8002).

4. Set the Method to POST

<img width="428" alt="image" src="https://github.com/DalgoT4D/dalgot4d.github.io/assets/2160416/37efc461-2767-4b08-bd71-a7651a3c6037">

5. Set the run states that we are interested in are Completed, Cancelled, Crashed, Failed, TimedOut

<img width="428" alt="image" src="https://github.com/DalgoT4D/dalgot4d.github.io/assets/2160416/318653b6-f0b1-4eec-baae-09ee729dc3d0">

6. Set the Headers. The notification key here should be the one set in Dalgo backend .env under PREFECT_NOTIFICATIONS_WEBHOOK_KEY

  `{"X-Notification-Key": "********"}`

<img width="418" alt="image" src="https://github.com/DalgoT4D/dalgot4d.github.io/assets/2160416/0a05bda6-e134-4b59-b68f-d43452e607b9">

7. Set the JSON Data to

  `{"body": "{{body}}"}`

<img width="378" alt="image" src="https://github.com/DalgoT4D/dalgot4d.github.io/assets/2160416/957571d3-f899-4bf4-a8bf-93b3c7d00953">

8. Save

9. Use the API GET /api/flow_run_notification_policies/{id} to make sure that this notification has this message_template:

    Flow run {flow_run_name} with id {flow_run_id} entered state {flow_run_state_name}

10. If it does not, you should update it using PATCH /api/flow_run_notification_policies/{id}

