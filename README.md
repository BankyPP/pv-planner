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
    function signedIn() { return request.auth != null; }
    function me() { return request.auth.uid; }

    // Your own planner — only you
    match /users/{uid}/planner/{docId} {
      allow read, write: if signedIn() && me() == uid;
    }

    // Public card (name, @username, photo) — any signed-in user can read, only you can write
    match /profiles/{uid} {
      allow read: if signedIn();
      allow write: if signedIn() && me() == uid;
    }

    // @username → uid (look up one exact name; can't list them)
    match /handles/{handle} {
      allow get: if signedIn();
      allow create: if signedIn() && request.resource.data.uid == me()
                    && handle.matches('^[a-z0-9_.]{3,20}$');
      allow delete: if signedIn() && resource.data.uid == me();
    }

    // exact email → uid, only for verified emails, only your own
    match /emails/{email} {
      allow get: if signedIn();
      allow create, update: if signedIn() && request.auth.token.email_verified == true
                            && email == request.auth.token.email.lower()
                            && request.resource.data.uid == me();
      allow delete: if signedIn() && resource.data.uid == me();
    }

    // Friend requests / friendships / blocks; id = smallerUid_largerUid
    match /friends/{pair} {
      allow read: if signedIn() && (resource == null || me() in resource.data.members);
      allow create: if signedIn()
        && me() in request.resource.data.members
        && request.resource.data.members.size() == 2
        && request.resource.data.members[0] < request.resource.data.members[1]
        && pair == request.resource.data.members[0] + '_' + request.resource.data.members[1]
        && ((request.resource.data.status == 'pending' && request.resource.data.from == me()
              && request.resource.data.to in request.resource.data.members && request.resource.data.to != me())
            || (request.resource.data.status == 'blocked' && request.resource.data.blockedBy == me()));
      allow update: if signedIn()
        && me() in resource.data.members
        && request.resource.data.members == resource.data.members
        && (resource.data.status != 'blocked' || resource.data.blockedBy == me())
        && ((request.resource.data.status == 'blocked' && request.resource.data.blockedBy == me())
            || (request.resource.data.status == 'accepted'
                && (resource.data.status == 'accepted' || (resource.data.status == 'pending' && resource.data.to == me())))
            || (request.resource.data.status == 'pending' && request.resource.data.from == me()));
      allow delete: if signedIn() && me() in resource.data.members
        && (resource.data.status != 'blocked' || resource.data.blockedBy == me());
    }

    // What an owner shares with one friend; id = ownerUid_viewerUid
    match /shares/{id} {
      allow read: if signedIn() && (resource == null
        || resource.data.owner == me()
        || (resource.data.viewer == me()
            && get(/databases/$(database)/documents/friends/$(resource.data.pair)).data.status == 'accepted'
            && me() in get(/databases/$(database)/documents/friends/$(resource.data.pair)).data.members));
      allow create, update: if signedIn() && request.resource.data.owner == me()
        && id == me() + '_' + request.resource.data.viewer;
      allow delete: if signedIn() && resource.data.owner == me();
    }

    // Shared tasks, event invites and challenges
    match /collab/{id} {
      allow read: if signedIn() && (resource == null || me() in resource.data.members);
      allow create: if signedIn() && request.resource.data.owner == me()
        && me() in request.resource.data.members && request.resource.data.members.size() <= 20;
      allow update: if signedIn() && me() in resource.data.members
        && request.resource.data.owner == resource.data.owner
        && (me() == resource.data.owner
            || request.resource.data.diff(resource.data).affectedKeys().hasOnly(['done', 'rsvp', 'prog', 'members', 'updatedAt']));
      allow delete: if signedIn() && resource.data.owner == me();
    }
  }
}
```

4. Paste the web app's `firebaseConfig` values into the `FIREBASE` constant in `index.html`

## Friends

Friends use top-level collections next to your private planner: `profiles`, `handles`, `emails`, `friends`, `shares` and `collab`. The rules above (also in `firestore.rules`) keep each user's planner private and only let friends read what the owner chose to share. Publish them in Firebase Console → Firestore Database → Rules.

## Google Calendar

- OAuth client: `300153420589-p4735rii5bcgnbv2g8ue1tsapa1cieag.apps.googleusercontent.com` (Authorized JavaScript origin `https://bankypp.github.io`)
- Scopes: `calendar.calendarlist.readonly`, `calendar.events.readonly`, `calendar.app.created`
- Settings → Google Calendar: connect, pick which calendars show in the planner
- Tasks with “Show in Google Calendar” are pushed to a “PV Planner” calendar the app creates (event id = `pv` + hex(task id)); each sync reconciles that calendar with the tasks
- Access tokens last ~1 hour; when expired the app shows a “Reconnect” banner
