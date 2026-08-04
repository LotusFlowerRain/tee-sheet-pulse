
# Tee Sheet Pulse
Real-time pace-of-play management for semi-private and university golf courses.
Live demo: https://tee-sheet-pulse.vercel.app
Self-test: append `?selftest=1` to the demo URL (40 assertions against the pace logic)
Architecture and data flow: [ARCHITECTURE.md](ARCHITECTURE.md)
Built for CEE 250 / MS&E 265 (Feature Flow), Stanford, Spring 2026.
Pro shop dashboard built with AI-assisted development (Claude Code);
hardware (GPS clip) designed in Fusion 360, not yet manufactured — demo runs
on simulated GPS and time-based projection.

The golfer-facing page has been dropped; the product is now the pro shop
board only, across three views:

- **Live Tee Sheet** — pace status per group, gap and cascade alerts, tracker
  assignment, and a legend covering On Pace / Watch / Behind / Within 10 min / Alert
- **Course Board** — illustrated course map with live group positions
- **Player History** — player database of recorded pace behaviour, used to brief
  the marshal for preventive action before a player's next round
