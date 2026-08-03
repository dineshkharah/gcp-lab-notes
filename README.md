# gcp lab notes

Notes from working through Google Cloud Skills Boost labs, mostly the challenge labs that sit at the end of a skill badge. The point of this repo is to not lose time to the same problem twice. Every lab that went wrong has the cause and the fix written down.

## How this is organised

- `workflow.md` is the process. How to run a lab from start to finish, what to line up before touching a command, what to check after each task.
- `gotchas.md` is the list of traps that turn up across many labs. Read this one first. Almost all the time lost in these labs came from a handful of repeating causes and they are all in there.
- `labs/` has one file per lab, named by its lab id. Each file lists the tasks, the commands that actually worked, and what went wrong.

## How to use it before a lab

1. Read `gotchas.md`.
2. Open the matching file in `labs/` if there is one.
3. Copy the values off the started lab page into the commands.

## The values in these notes are not answers

Every lab hands out fresh names when it starts. Region, zone, project id, bucket name, topic name, table name, file names, all of it changes per run. The values written here are from one old run and are only there to show the shape of the command. Read your own off the lab page.
