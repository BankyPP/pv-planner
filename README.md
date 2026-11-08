# Personal Planner (pv-planner)

Personal task and life-planning web app — Dashboard, Daily Planner, Weekly Planner, Goals (with milestones), Weekly Review, and Settings.

- Site: https://bankypp.github.io/pv-planner/
- Sign-in: Google or email & password, via the Firebase project `pv-planner-96e2b`
- Data: Firestore at `users/{uid}/planner/{docId}` — one document per task, goal, weekly focus, and weekly review, plus `profile`
- Everything lives in a single `index.html` — no build step

## Firebase setup

1. Authentication → Sign-in method: enable **Google** and **Email/Password**
2. Authentication → Settings → Authorized domains: add `bankypp.github.io`
3. Firestore Database → Rules:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{uid}/planner/{docId} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

4. Paste the web app's `firebaseConfig` values into the `FIREBASE` constant in `index.html`

## Google Calendar

- OAuth client: `300153420589-p4735rii5bcgnbv2g8ue1tsapa1cieag.apps.googleusercontent.com` (Authorized JavaScript origin `https://bankypp.github.io`)
- Scopes: `calendar.calendarlist.readonly`, `calendar.events.readonly`, `calendar.app.created`
- Settings → Google Calendar: connect, pick which calendars show in the planner
- Tasks with “Show in Google Calendar” are pushed to a “PV Planner” calendar the app creates (event id = `pv` + hex(task id)); each sync reconciles that calendar with the tasks
- Access tokens last ~1 hour; when expired the app shows a “Reconnect” banner
