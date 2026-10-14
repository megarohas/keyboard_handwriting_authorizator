# Keyboard Handwriting Authorizator

A web app that logs you in by *how* you type, not only by what you type:
keystroke timings of a check phrase are fed to a small neural network that
tries to recognise the user.

## How it works

1. **Sign up** with name, email and password (bcrypt-hashed, stored in
   MongoDB). Every user gets a sequential numeric id.
2. **Check phrase.** All users type the same phrase. It belongs to the
   most recently created network and is set through `POST /api/update_phrase`,
   which also creates a fresh, untrained network for it.
3. **Keystroke features.** While you type the phrase, every `keydown`/`keyup`
   is recorded. `helpers/front/funcs.js` turns them into a vector: hold time of
   each key, the gap between releasing one key and pressing the next, the total
   typing time (all scaled into `[0, 1)`), plus Shift and Caps Lock flags.
4. **Training.** On the dashboard you type the phrase again and press *Train*.
   The vector is added as a sample to a [brain.js](https://github.com/BrainJS/brain.js)
   feed-forward network (one hidden layer of 3 neurons, leaky ReLU) with the
   user's id as the target output. Every run is saved and listed on the dashboard.
5. **Smart login.** On the login form, if you typed into the phrase field, the
   app skips the password: it runs the network on your timings, and if the
   output lands close enough to a whole number, rounds it and compares it with
   the id of the account for the email you entered. A match issues a JWT.
   If the phrase field was left untouched, a normal email + password login is used.

## Tech stack

- Next.js 9 (pages + API routes), React 16, Material-UI 4
- MongoDB via Mongoose (`users`, `nets`, `trains` collections)
- brain.js 2 (alpha), bcrypt, jwt-simple, react-cookie

## API routes

| Route | Purpose |
| --- | --- |
| `POST /api/sign_up` | Create an account |
| `POST /api/log_in` | Email + password login, returns a JWT |
| `POST /api/smart_log_in` | Email + keystroke vector login, returns a JWT |
| `POST /api/train` | Add a keystroke sample for a user and save the run |
| `GET /api/get_current_phrase` | Current check phrase |
| `POST /api/update_phrase` | Set a new check phrase and start a new network |
| `POST /api/get_current_user` | User for a token |
| `POST /api/get_user_trains` | Training history of a user |
| `GET /api/ping` | JWT check used by protected pages |

`/api/brain`, `/api/create_user` and `/api/get_users` are leftover
experiments and debug helpers.

## Getting started

The project dates from 2019 and uses `node-sass` 4 and `bcrypt` 3, which need
native builds — use Node 12.

1. Point the app at your own database: the MongoDB connection string is
   hard-coded in `helpers/back/mongo_connector.js`. The JWT secret is also
   hard-coded in the API routes.
2. Install and run:

   ```bash
   yarn install
   yarn dev            # http://localhost:3000
   ```

3. Create the first check phrase (login and training fail until a network exists):

   ```bash
   curl -X POST http://localhost:3000/api/update_phrase \
     -H 'Content-Type: application/json' \
     -d '{"phrase": "The quick brown fox jumps over the lazy dog."}'
   ```

4. Sign up, log in with your password, and train the network a few times on the
   dashboard. Then log out and try logging in by typing the phrase only.

Production build: `yarn build && PORT=3000 yarn start`.

## Project structure

```
pages/            index (login / sign up), dashboard, API routes
components/       login and sign-up forms, phrase input analyzer, profile, training list
helpers/front/    keystroke feature extraction, SSR auth check
helpers/back/     MongoDB connection wrapper, JWT check
db/schema.js      Mongoose schemas
```

## Context

University project at Omsk State University (OmSU), built in December 2019 –
January 2020. It is a proof of concept, not production authentication: there is
one shared network for all users, it is retrained on one sample at a time, and
secrets live in the source code.
