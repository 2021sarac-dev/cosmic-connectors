# 🔭 Cosmic Connectors

**Unlocking the Cosmos for Classrooms**

Cosmic Connectors connects teachers with astronomical societies, universities and space enthusiasts to share telescopes and expertise. Underfunded (Title 1) schools get priority, so that every student gets to explore space.

Built for the **2026 Congressional App Challenge**.

---

## Table of contents

- [The problem](#the-problem)
- [Who it's for](#who-its-for)
- [Features](#features)
- [Tech stack](#tech-stack)
- [Architecture](#architecture)
- [Data model](#data-model)
- [How driving distance works](#how-driving-distance-works)
- [Key code](#key-code)
- [Project structure](#project-structure)
- [Running the app](#running-the-app)
- [Known limitations](#known-limitations)
- [Roadmap (version 2)](#roadmap-version-2)
- [Use of AI and outside tools](#use-of-ai-and-outside-tools)
- [Author](#author)

---

## The problem

Students learn the most about space by looking through a real telescope, but telescopes are expensive and many schools can't afford them. Meanwhile, astronomy clubs and universities have portable telescopes and expert volunteers who are free on weeknights. The gap is widest at Title 1 schools, which serve the most students from low-income families.

## Who it's for

| Role | Who they are | What they do in the app |
|---|---|---|
| **Explorer** | Science teachers, school staff, education leaders | Request a telescope or a guest speaker for a class event |
| **Sky Partner** | Astronomy clubs, university labs, space enthusiasts | Review requests from schools and confirm a match |
| **Admin** | The app owner | View all accounts and all requests |

## Features

- **Role-based sign-up.** Explorers and Sky Partners register with name, organization and a full street address (street, city, state, ZIP). Explorers mark whether their school is Title 1.
- **Sign in.** A dedicated Sign In page sends each user to the right dashboard.
- **Requests.** Explorers request a *Telescope* or a *Guest speaker* with a topic, date, time and optional comments.
- **Title 1 priority.** On the Sky Partner's Incoming Requests page, Title 1 requests appear first in a labelled, highlighted **★ Priority** group. Within each group, the soonest event comes first.
- **Driving distance.** Each request shows the driving miles and minutes between the school and the Sky Partner.
- **The handshake.** A Sky Partner clicks **Confirm Match**. The app checks again before saving, so two partners can never confirm the same request.
- **Confirmation email.** After confirming, the partner gets a pre-filled email (To, Subject and message) ready to send to the teacher.
- **Dashboards.**
  - Explorers see each request's status ("Waiting for a Sky Partner" or "Matched with …").
  - Sky Partners see the requests they've committed to.
  - The admin sees everything.
- **Clean, responsive design.** A light, classic look with a two-column welcome page that stacks to one column on phones.

## Tech stack

| Layer | Technology |
|---|---|
| Languages | HTML5, CSS3, JavaScript (ES2020: `async`/`await`, template literals, spread syntax) |
| Frameworks | None: plain ("vanilla") JavaScript, no build step |
| Styling | CSS custom properties (variables), Flexbox, media queries, Google Fonts (Fraunces, Inter) |
| Hosting | Claude Artifacts (claude.ai). The app is published as a single web page. |
| Database | Claude Artifact shared database (`db` capability): JSON documents in collections, shared by all users |
| Image storage | Claude Artifact asset store (`assets` capability) |
| Geocoding | U.S. Census Geocoder (address → latitude/longitude); Zippopotam.us (ZIP-code fallback) |
| Routing | OSRM (Open Source Routing Machine) on OpenStreetMap roads, for driving distance and time |
| Automation | Claude scheduled task (runs hourly) that fills in missing distances |
| Email | `mailto:` link that opens the user's own email app |
| Development tools | Claude (AI coding coach), Chromium browser for testing, Python + Pillow/OpenCV for cropping images |

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  LAYER 1 · PRESENTATION  (HTML + CSS, in the browser)        │
│  Welcome · Sign In · Registration · Explorer dashboard ·     │
│  New Request · Sky Partner dashboard · Incoming Requests ·   │
│  Admin                                                       │
└───────────────────────────────┬──────────────────────────────┘
                                │ button clicks call functions
┌───────────────────────────────▼──────────────────────────────┐
│  LAYER 2 · APPLICATION LOGIC  (vanilla JavaScript)           │
│  showScreen · signIn · createExplorerAccount ·               │
│  createPartnerAccount · sendRequest · loadIncomingRequests · │
│  confirmMatch · addDistances · escapeHtml                    │
└───────────────────────────────┬──────────────────────────────┘
                                │ claude.use("db") · get / set
┌───────────────────────────────▼──────────────────────────────┐
│  LAYER 3 · PLATFORM  (Claude Artifact hosting, sandboxed)    │
└───────────────────────────────┬──────────────────────────────┘
                                │ JSON documents
┌───────────────────────────────▼──────────────────────────────┐
│  LAYER 4 · PERSISTENCE                                       │
│  accounts · requests · distances   +   image asset store     │
└───────────────────────────────▲──────────────────────────────┘
                                │ writes new distances (hourly)
┌───────────────────────────────┴──────────────────────────────┐
│  BACKGROUND JOB · Claude scheduled task                      │
│  Census Geocoder → (ZIP fallback) → OSRM routing             │
└──────────────────────────────────────────────────────────────┘
```

All screens and logic run in the visitor's browser. Every account, request and distance is saved to the shared database, so every user sees the same data.

## Data model

Three collections of JSON documents. Each document has a readable id, so the app can load it directly without searching.

**`accounts`** (id = the user's email)

| Field | Notes |
|---|---|
| `name` | Person's name |
| `role` | `explorer`, `sky_partner` or `admin` |
| `school` / `organization` | School name (Explorer) or club/university (Sky Partner) |
| `street`, `city`, `state`, `zip`, `address` | Split address plus one combined line |
| `title1` | `true` if the school is Title 1 (Explorers only) |
| `password` | ⚠️ Stored as plain text in this prototype; see limitations |

**`requests`** (id = `"req_" + timestamp`)

| Field | Notes |
|---|---|
| `explorerEmail`, `school`, `title1` | Copied from the Explorer's account |
| `type` | `Telescope` or `Guest speaker` |
| `topic`, `date`, `time`, `comments` | What the teacher entered |
| `status` | `pending` or `confirmed` |
| `confirmedBy` | Email of the Sky Partner who confirmed |

**`distances`** (id = `explorerEmail + "__" + partnerEmail`)

| Field | Notes |
|---|---|
| `miles`, `minutes` | Driving distance and time |
| `approximate` | `true` when a ZIP-code centre was used; shown with "~" |

## How driving distance works

The page runs in a sandbox that **cannot call outside websites**. A test page calling the map services directly failed with `TypeError: Failed to fetch`. So distances are **precomputed** and stored:

1. An **hourly scheduled task** finds school and partner pairs with no saved distance.
2. The **U.S. Census Geocoder** turns each street address into latitude and longitude. If it can't match the street, the **ZIP-code centre** from Zippopotam.us is used instead, and the result is marked approximate.
3. **OSRM** returns the driving miles and minutes between the two points.
4. The result is saved to the `distances` collection.
5. When a Sky Partner opens Incoming Requests, `addDistances()` reads the saved values and fills in the **Drive** column.

**Approaches tried before this one**
- Asking an AI model to estimate distance. It needed a permission prompt every time and often came back blank.
- Calling the map services from the page. The sandbox blocked it.
- The Census geocoder alone. Some streets didn't match, so the ZIP fallback was added.

## Key code

**Title 1 priority + soonest-first sorting** (`loadIncomingRequests`)

```javascript
// sort every request by date and time ("2026-10-15 18:30" sorts correctly as text)
let openDocs = snapshot.docs.slice().sort(function (a, b) {
  let whenA = (a.data().date || "") + " " + (a.data().time || "");
  let whenB = (b.data().date || "") + " " + (b.data().time || "");
  return whenA < whenB ? -1 : (whenA > whenB ? 1 : 0);
});

// pass 1: Title 1 open requests, highlighted
for (let i = 0; i < openDocs.length; i++) {
  let req = openDocs[i].data();
  if (req.status === "pending" && req.title1 === true) {
    title1List += buildRequestLine(req, openDocs[i].id, known[openDocs[i].id], true);
  }
}
// pass 2: every other open request
for (let i = 0; i < openDocs.length; i++) {
  let req = openDocs[i].data();
  if (req.status === "pending" && req.title1 !== true) {
    otherList += buildRequestLine(req, openDocs[i].id, known[openDocs[i].id]);
  }
}
```

**The handshake lock** (`confirmMatch`)

```javascript
let req = { ...doc.data() };          // saved data is frozen, so copy it first
if (req.status === "confirmed") {      // re-check right before saving
  // someone else got there first, so stop
  return;
}
req.status = "confirmed";
req.confirmedBy = currentUser.email;
await db.collection("requests").doc(requestId).set(req);
```

**Safety:** all user-typed text passes through `escapeHtml()` before it is placed on the page, so nobody can inject HTML or scripts through a form.

## Project structure

```
cosmic-connectors/
├── index.html        # the whole app: HTML, CSS and JavaScript
├── images/
│   ├── home.jpg      # welcome page cover image
│   ├── roles.jpg     # "Choose your role" picture
│   ├── signin.jpg    # Sign In page
│   ├── register.jpg  # registration pages
│   └── waiting.jpg   # empty request list
├── README.md
└── README.txt
```

## Running the app

**Live app:** the app runs as a Claude artifact on claude.ai, where the shared database is available. This is the version to use and demo.

**Local copy:** open `index.html` in a browser to view the pages, layout and images. Signing in, registering and sending requests **will not work locally**, because they depend on the artifact's shared database (`claude.use("db")`), which only exists on claude.ai.

## Known limitations

- ⚠️ **Passwords are stored as plain text.** This is acceptable for a prototype with test accounts only, and must be fixed before real use (see roadmap).
- Each list loads the whole `requests` collection and filters in the browser. That is fine for dozens of requests and slow for thousands.
- New accounts can wait up to an hour for their driving distance.
- The 50-mile matching radius from the original plan is **shown but not yet enforced**.
- The confirmation email is a draft the partner must send; the app cannot send email by itself.
- Sky Partners can't yet list their telescopes or speaking topics (the "marketplace").

## Roadmap (version 2)

1. **Secure sign-in:** managed authentication with hashed passwords and email verification; move data to PostgreSQL behind an API.
2. **Location built in:** geocode at registration, store locations with PostGIS, and enforce the 50-mile radius with one spatial query.
3. **Automatic alerts:** a message queue that sends confirmation emails and new-request alerts automatically.
4. **The marketplace:** Sky Partners list telescopes, talk topics and available dates; matching uses them.
5. **Mobile and impact:** a mobile app, plus reports on students reached at Title 1 schools by district.

## Use of AI and outside tools

In line with the Congressional App Challenge rules, all AI use is disclosed here:

- **Claude (Anthropic)** was used as an AI coding coach. It explained concepts, suggested and reviewed code, helped find and fix bugs, and runs the hourly scheduled task that computes driving distances.
- **Some illustrations were AI-generated** (the cover image was labelled "Made with AI").
- **My own work:** the idea and the written plan, every feature and design decision, testing with multiple accounts, finding bugs, and directing each change. *(Edit this section so it accurately describes your own contributions.)*
- **Outside services:** U.S. Census Geocoder, Zippopotam.us, OSRM / OpenStreetMap and Google Fonts.

## Author

**[Your name]**, [school], [grade]

Inspired by my interest in astrobiology, my time as president of my middle school astronomy society, and the volunteers of the East Bay Astronomical Society who shared their time and telescopes with me.
