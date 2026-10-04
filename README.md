# Study Tracker

A desk study tracker for a first-year Electronics and Communication student, built around a Raspberry Pi 3 Model B+ and a Google AIY Voice Kit V1.

**Current stage: project design, architecture, and OS preparation.** The tracker application and dashboard are planned; they are not implemented yet.

## Project design and diagrams

Read the [project document](.scratch/beginner-friendly-python-productivity/project.md) for the complete first-version idea and requirements. The [architecture document](.scratch/beginner-friendly-python-productivity/architecture.md) contains Mermaid diagrams for the system, Python modules, button state machine, gestures, event flow, audio, and dashboard.

The [data and recovery design](.scratch/beginner-friendly-python-productivity/data-and-recovery.md) diagrams SQLite records, timing, checkpoints, midnight splitting, and interrupted-session recovery. The [publication architecture](.scratch/beginner-friendly-python-productivity/publication-architecture.md) diagrams snapshots, offline retries, deployment serialisation, and the visibility gate.

GitHub renders the Mermaid blocks directly. The designs identify proposed defaults and unresolved choices; they do not claim implementation or hardware verification.

## Start here

Follow [Install Raspberry Pi OS and prerequisites](.scratch/beginner-friendly-python-productivity/setup.md). It takes you from a blank microSD card to a Pi ready for development.

Then follow [Verify the Voice HAT V1](.scratch/beginner-friendly-python-productivity/voice-hat.md) to check the speaker and physical button. [Prepare Cloudflare publishing](.scratch/beginner-friendly-python-productivity/publishing.md) covers the pinned deployment tools and the laptop fallback.

Read [Repository and contribution guide](.scratch/beginner-friendly-python-productivity/repository.md) before adding files. [Verification record](.scratch/beginner-friendly-python-productivity/verification.md) distinguishes checks completed here from checks still needed on the Pi.

## What we are building

Double-tap the button to start or end a session. Single-tap to take a break or resume studying. Distinct short tones confirm actions. SQLite stores events locally, and Python generates a mobile-friendly dashboard for Cloudflare Pages.

The Pi should work offline. The website will show the last published snapshot and remain available when the Pi is off. There will be no scheduled session endings, AI, extra sensors, frontend framework, or cloud database.

Python will handle tracking, hardware, storage, tone generation, dashboard generation, and deployment orchestration. HTML, CSS, and a little JavaScript will handle the browser interface. Node.js is only needed to run Cloudflare's Wrangler deployment CLI.

## Public source, personal data

This repository is public. Keep your actual study database, exported history, generated dashboard, logs, Wi-Fi details, and Cloudflare token out of Git. The ignore rules help, but review files before committing.

Publishing the source does not decide who can view the future dashboard. We will preview it and choose public or properly protected access before its first publication.

## Next step after setup

Once the OS, button, and audio checks are complete, continue the decision map and then build through simulation/storage, hardware/tones, dashboard, publishing/retries, and startup/recovery documentation. The wider project remains in planning; this preparation step does not implement it.
