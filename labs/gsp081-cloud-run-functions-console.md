# GSP081, Cloud Run functions qwik start, console

Small console lab. Create a function, deploy it, test it, look at logs. Ten minutes.

Worth doing in the console rather than the shell, because the inline editor hands you a working `helloHttp` function for free.

## Task 1

Cloud Run, Services, Write a function.

- Service name `gcfunction`
- Region from the lab page
- Authentication: Allow public access
- Under Containers, Volumes, Networking, Security: execution environment **Second generation**, maximum number of instances **5**
- Memory stays default

Click Enable if a popup asks about apis.

## Task 2

Leave the default `index.js` alone and click Save and redeploy. Wait for the spinner to become a green check.

The default code reads `req.query.name || req.body.name || 'World'`, so it answers `Hello World!` no matter what you put in the test body. That is expected.

## Task 3

Click Test, set the triggering event body to `{"message":"Hello World!"}`, then copy the generated cli command and run it in Cloud Shell.

The lab says to enter the text between the brackets, meaning the field should end up holding the full object.

## Task 4

Service Details, Observability, Logs.

## Quiz

- Cloud Run functions is a serverless execution environment for event driven services: **True**
- Trigger type used in this lab: **HTTPS**
