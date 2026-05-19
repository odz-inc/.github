# odz-inc

Building **JustTrip** — an AI-powered trip planner that adapts to you, not the other way around.

## What is JustTrip?

Planning a trip is fragmented: you check weather on one app, search places on another, figure out packing yourself, and juggle bookings across three tabs. JustTrip collapses that into one flow.

Tell it where you're going, how many days, and what kind of trip — nature getaway, group nightlife, solo relaxation, or anything in between. It generates a full day-by-day itinerary, checks the forecast, suggests what to pack, and connects to flights and accommodation. Edit the plan and it learns your taste for next time.

**Closed beta. iOS + Android.**

## What makes it different

- **Personalization that learns** — every edit you make (swap a place, reorder a day) is used to tune your preference profile. The more you use it, the better it knows you.
- **Weather-aware scheduling** — outdoor vs. indoor activities are weighted against the actual forecast for your dates.
- **Packing intelligence** — destination and trip style specific, covering 15+ countries.
- **Visa & travel rules** — built-in document requirements for 20+ country pairs.
- **Flights & booking** — not just itineraries, connected to the actual trip logistics.

### With a subscription: your travel partner, not just a planner

Free tier gets you the plan. Paid tier keeps working once you're already there.

If a place is unexpectedly closed, overcrowded, or just not what you expected — JustTrip surfaces alternatives on the spot so you can adapt without losing momentum. No scrambling through Google, no wasted time. The trip adjusts around reality, not the other way around.

## Why we're building this

As a backend and integration developer, I've spent years working across fragmented data sources. The travel space is a perfect example of the broader problem: the data already exists — weather forecasts, place ratings, visa rules, flight availability, packing norms — it's just scattered across dozens of services with no shared logic connecting them.

The world doesn't need more data. It needs smarter integration and the ability to move fluidly across what already exists. JustTrip is built on that belief: one coherent flow, pulling from the right sources at the right time, personalized to the person making the trip.

## Tech

FastAPI · PostgreSQL + pgvector · Gemini 2.5 Flash · Vertex AI · Flutter · Firebase · Google Cloud Run

See [JustTrip/architecture.md](JustTrip/architecture.md) for the full technical overview.

## Team

- Bartłomiej Ochodek — Founder / Fullstack dev
- Hubert Bojda — DevOps / DevSecOps
- Jan Górecki — Designer and Tester
