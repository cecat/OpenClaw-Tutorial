# OpenClaw-Tutorial - WIP

**CAUTION: This repo is in the midst of being overhauled, as new versions of OpenClaw require significant changes to our scaffolding.  More importantly, the overhaul will migrate to NemoClaw+Hermes as a second experiment in this WIP.   There are many things to learn in this repo, but I would advise against trying to reproduce the system.**

This project deploys OpenClaw with a focus on (a) safety, security, and alignment, (b) reliability and predictability, and (c) flexible system management.

The repo is organized as a tutorial, which covers the big picture, our objectives, and how we want about deploying OpenClaw to address those objectives.

## Contents of this Repo

- [Tutorial](https://github.com/cecat/OpenClaw-Tutorial/blob/main/OpenClaw-Tutorial.md) - a very long overview of OpenClaw and the scaffolding we've built. For our purposes and objectives, we found it useful to deveoop a handful of *Enhancements*, each addressing something we needed OpenClaw (or one or more claws) to be able to do.  We also discovered some rabbit holes that we had to climb out of, and kept track of these in a *Lessons Learned* appendix.

- [Quickstart](https://github.com/cecat/OpenClaw-Tutorial/blob/main/Quickstart.md) -  I was tempted to delete (see **CAUTION** above) but am leaving this here as some of it may still be helpful as an interesting example.

- [Gateway](https://github.com/cecat/OpenClaw-Tutorial/tree/main/gateway) - things you need to fire up an OpenClaw gateway (including docker-compose and various config files - see Quickstart).

- [Integrations](https://github.com/cecat/OpenClaw-Tutorial/tree/main/Integrations) - Various step-by-step for adding capabilities like Slack, Google Drive, and Gmail/Calendar.
