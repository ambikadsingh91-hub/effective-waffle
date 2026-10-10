# LOT Ecosystem — Firebase starter implementation

This project unifies LOT Shop, VPay, and VStocks under one Firebase Authentication identity and one Firestore `accounts/{uid}.balance` record. It reuses the existing LOT Shop dark UI as the styling baseline and adds wallet, simulated trading, orders, and admin areas.

## Important status

This is a deployable **starter implementation**, not a claim that every requested production feature is finished or deployed. Firebase has not been tested from this environment. The core callable functions implemented here cover account profile registration, username login lookup, listing creation/updates, purchases, LOT transfers, order status changes, simulated trades, one-time Virat admin bootstrap, balance adjustments, and admin password resets. Before public launch, finish and test the refund/dispute flow, seller order UI, webhook delivery via a vetted queue/egress service, full audit coverage, pagination, rate limiting/App Check, and a server-side simulated price updater. Do not advertise those unfinished features as working.

## Firebase Console setup

1. Open Firebase project `lotapimain` and enable **Authentication → Email/Password**.
2. Create the **Cloud Firestore** database in production mode. Do not enable Realtime Database for this project.
3. Enable **Storage** and review billing/region settings.
4. Add your Hosting domain to Authentication authorized domains.
5. Install Node.js 20+ and Firebase CLI: `npm install -g firebase-tools`.
6. Copy `.firebaserc.example` to `.firebaserc`, verify the project ID, then run `firebase login`.
7. Install Functions dependencies: `cd functions && npm install && cd ..`.
8. Set the one-time admin bootstrap secret: `firebase functions:secrets:set LOT_ADMIN_BOOTSTRAP_SECRET`.
9. Deploy rules/functions/hosting: `firebase deploy`.

## First administrator bootstrap (securely)

1. Register the username `Virat` through the signup page. Keep its password private and strong.
2. From a trusted operator-only environment, invoke `bootstrapViratAdmin` once with the secret configured in Secret Manager. Do **not** put the secret into the public website or commit it. The initial account must have username `Virat`; the function refuses bootstrap if another admin already exists. The browser UI intentionally does not expose this bootstrap call. Use a trusted operator script with Firebase client auth and callable functions, then force-refresh the Virat ID token/sign in again.
3. Verify the `admin` custom claim and audit record before using admin tools. Do not create admin status by editing a client profile.

## Username login design

Firebase Email/Password Authentication is used with an internal synthetic address of the form `username@users.lotapimain.invalid`; users enter only username and password. The callable profile-registration operation reserves the lowercase username transactionally. This is not a user email and password reset by email is intentionally absent. The UI does not display or request an email.

## Architecture

- `public/index.html`, `public/style.css`, `public/ecosystem.css`: existing LOT Shop design baseline plus integrated sections.
- `public/app.js`: Firebase modular SDK, auth, Firestore reads, Storage image uploads, callable function calls.
- `functions/index.js`: trusted server-side operations; never trust client prices or balances.
- `firestore.rules`, `storage.rules`: default-deny direct writes for protected records.
- `accounts/{uid}` is the only authoritative LOT balance.
- `ledger/{id}` is append-only from clients' perspective. A callable operation creates ledger records and changes balances in one Firestore transaction.
- `operations/{uid_operationId}` stores idempotency results for purchases, transfers and trades.
- Stock quotes in `marketPrices/{TICKER}` must be seeded and updated by a trusted server process. Until that is done, VStocks shows clearly labelled sample simulator cards in the UI, but trades will correctly fail because no server-authoritative quote exists.

## Firestore indexes

Firestore may prompt you to create composite indexes for queries on `ledger` (`participantUids` array-contains + `createdAt` descending), `orders` (`buyerUid` + `createdAt` descending), and `trades` (`uid` + `createdAt` descending). Create only the indexes Firebase requests.

## Run locally

```bash
npm install -g firebase-tools
firebase login
cd functions && npm install && cd ..
firebase emulators:start --only hosting,functions,firestore,auth,storage
```

Serve the public files through Firebase Hosting (not by opening `index.html` as a `file://` URL). Use the Firebase Emulator Suite first for testing. Then deploy with `firebase deploy`.

## Known gaps to finish before production

- Implement and test refund/cancellation/dispute workflows with stock restoration and immutable ledger entries.
- Add seller order-management UI and server-validated Processing → Delivered transitions.
- Add a trusted simulated-market price seeder/updater and market pause settings.
- Add seller webhook configuration encrypted/private on the backend and deliver via a queue with URL allow-list/SSRF protection. Arbitrary Discord webhook URLs are deliberately not called from the browser or this starter function.
- Add Firebase App Check, rate limits, abuse monitoring, structured operational logs, and a reconciliation job.
- Add full pagination and broader audit coverage. Avoid allowing users to create unbounded transaction volume.
- Test username account-creation cleanup for Auth accounts whose profile reservation fails; production onboarding should use a trusted callable provisioning flow or a cleanup queue.
- For higher traffic, shard operational counters and reconsider Firestore transaction hot spots. Firestore transactions have finite document/read/write limits and may retry under contention.

## Currency and fees

LOT is virtual-only. Marketplace fee is currently zero: purchase total is debited from the buyer and credited to the seller atomically. Stock trades debit LOT when buying and credit LOT when selling at the current simulated server quote; this simplified simulator has no trading fee. Do not add deposits, withdrawals, or cash redemption without a separate design and explicit request.
