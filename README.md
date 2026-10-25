# Uber-fullstack-clone

Uber Video is a full-stack ride-booking web application with separate experiences for riders and captains (drivers). Riders can choose pickup and destination locations, compare estimated fares for different vehicle types, and request rides. Captains can share their location, receive ride requests in real time, accept a request, verify a rider with an OTP, and complete the ride.

The project is organized as two applications inside `uber-fullstack-clone/`:

- `uber-fullstack-clone/frontend` — React single-page application built with Vite.
- `uber-fullstack-clone/Backend` — Express API, MongoDB data models, and Socket.IO server.

## Key highlights

- **Two account types:** rider registration/login and captain registration/login, including captain vehicle details.
- **Protected pages and API routes:** signed JWTs identify users and captains; logged-out tokens are blacklisted.
- **Location lookup:** Google Maps Platform services provide address suggestions, geocoding, distance, and travel-time data.
- **Fare estimates:** the API calculates an estimate for car, moto, and auto rides from route distance and duration.
- **Live map:** the frontend displays a Google Map and the browser's current location.
- **Real-time ride updates:** Socket.IO notifies nearby connected captains about ride requests and sends status updates to riders.
- **OTP ride start:** a six-digit OTP is generated for each ride and must match before a captain can start it.
- **Responsive ride panels and transitions:** Tailwind CSS and GSAP animate the booking and captain workflow panels.

## Technology stack

| Area | Technologies |
| --- | --- |
| Frontend | React 18, Vite, React Router, Axios |
| UI and animation | Tailwind CSS, GSAP, `@gsap/react`, Remix Icon |
| Maps | `@react-google-maps/api`, Google Maps Platform APIs |
| Real-time client | Socket.IO Client |
| Backend | Node.js, Express 4, Socket.IO |
| Database | MongoDB with Mongoose |
| Authentication | JSON Web Tokens, bcrypt password hashing, token blacklist |
| Validation | express-validator |

## How the application works

### Rider flow

1. A rider registers or logs in. The backend validates the input, hashes passwords with bcrypt, and returns a JWT.
2. The rider enters pickup and destination addresses. The frontend asks the backend for Google Places autocomplete suggestions.
3. The rider requests a fare estimate. The backend uses Google distance and duration data to estimate fares for the supported vehicle categories.
4. The rider selects a vehicle and creates a ride. The backend stores a pending ride with a generated six-digit OTP.
5. The backend looks for captains near the pickup location and sends a `new-ride` Socket.IO event to connected captains.
6. When a captain accepts, the rider receives a `ride-confirmed` event. The captain enters the OTP to start the ride.
7. On a successful OTP check, the backend changes the ride status to ongoing and sends a `ride-started` event. When the captain finishes, the rider receives `ride-ended`.

### Captain flow

1. A captain registers with their name, email, password, and vehicle information, or logs in to an existing account.
2. On the captain home page, the frontend joins the captain's Socket.IO session and periodically sends browser geolocation updates to the backend.
3. Ride requests are delivered as `new-ride` events. The captain can accept a request, which assigns the captain to the ride and notifies the rider.
4. The captain verifies the rider-provided OTP to start the ride, then marks the ride complete when finished.

### Ride statuses

The ride model supports `pending`, `accepted`, `ongoing`, `completed`, and `cancelled` statuses. The current implemented workflow creates a pending ride, accepts it, starts it after OTP validation, and marks it completed.

## Project structure

```text
uber-video/
├── Backend/
│   ├── controllers/       # HTTP request handlers
│   ├── db/                # MongoDB connection
│   ├── middlewares/       # User and captain authentication
│   ├── models/            # User, captain, ride, and blacklist schemas
│   ├── routes/            # Express route definitions
│   ├── services/          # Account, ride, and maps logic
│   ├── app.js             # Express middleware and route registration
│   ├── server.js          # HTTP and Socket.IO server startup
│   └── socket.js          # Real-time events and socket helpers
└── frontend/
    └── src/
        ├── components/    # Booking, ride, map, and captain UI
        ├── context/       # Rider, captain, and Socket.IO state
        └── pages/          # Routes and application screens
```

## Requirements

- Node.js and npm.
- A MongoDB database (local or hosted, such as MongoDB Atlas).
- A Google Maps Platform API key with the APIs used by this app enabled:
  - Maps JavaScript API (frontend map display).
  - Places API (address autocomplete).
  - Geocoding API.
  - Distance Matrix API.
- Browser location permission for location tracking. For deployed sites, geolocation requires a secure context (HTTPS).

## Configuration

Create or update the environment files below. Keep real credentials private and do not commit `.env` files.

### `uber--fullstack-clone/Backend/.env`

```dotenv
DB_CONNECT=mongodb://127.0.0.1:27017/uber-video
JWT_SECRET=replace-with-a-long-random-secret
GOOGLE_MAPS_API=replace-with-your-server-side-google-maps-key
PORT=3000
```

- `DB_CONNECT` — MongoDB connection string.
- `JWT_SECRET` — secret used to sign and verify authentication tokens.
- `GOOGLE_MAPS_API` — server-side key for geocoding, distance, and place autocomplete requests.
- `PORT` — API and Socket.IO server port (optional; defaults to `3000`).

For MongoDB Atlas, use the connection string provided by Atlas in `DB_CONNECT`. Restrict the server-side Google key to the needed APIs and suitable server restrictions.

### `uber-fullstack-clone/frontend/.env`

```dotenv
VITE_BASE_URL=http://localhost:3000
VITE_GOOGLE_MAPS_API_KEY=replace-with-your-browser-google-maps-key
```

- `VITE_BASE_URL` — backend origin used by HTTP requests and the Socket.IO client.
- `VITE_GOOGLE_MAPS_API_KEY` — browser key used to load the map. Because `VITE_` variables are included in the browser bundle, this key is public; restrict it by HTTP referrer and API.

Restart the Vite development server after changing frontend environment variables.

## Install and run locally

Open two terminals from the directory containing the `uber-video` folder.

### 1. Start the backend

```powershell
cd uber-video\Backend
npm install
node server.js
```

The backend listens on port `3000` by default. If you have nodemon installed, you can use `nodemon server.js` instead for automatic restarts.

### 2. Start the frontend

```powershell
cd uber-video\frontend
npm install
npm run dev
```

Open the local URL printed by Vite (usually `http://localhost:5173`). Make sure `VITE_BASE_URL` points to the running backend.

### Production frontend build

From `uber-fullstack-clone\frontend`:

```powershell
npm run build
npm run preview
```

The backend package currently does not define a production or development start script; start it with `node server.js`.

## API overview

All endpoints are served from the backend origin. Protected endpoints accept the authentication token as a bearer token (the frontend currently sends it in the `Authorization` header).

| Method | Endpoint | Access | Purpose |
| --- | --- | --- | --- |
| `POST` | `/users/register` | Public | Register a rider |
| `POST` | `/users/login` | Public | Authenticate a rider |
| `GET` | `/users/profile` | Rider | Get the authenticated rider profile |
| `GET` | `/users/logout` | Rider | Log out and blacklist the rider token |
| `POST` | `/captains/register` | Public | Register a captain and vehicle |
| `POST` | `/captains/login` | Public | Authenticate a captain |
| `GET` | `/captains/profile` | Captain | Get the authenticated captain profile |
| `GET` | `/captains/logout` | Captain | Log out and blacklist the captain token |
| `GET` | `/maps/get-suggestions?input=...` | Rider | Get address suggestions |
| `GET` | `/maps/get-coordinates?address=...` | Rider | Convert an address to coordinates |
| `GET` | `/maps/get-distance-time?origin=...&destination=...` | Rider | Get route distance and duration |
| `GET` | `/rides/get-fare?pickup=...&destination=...` | Rider | Calculate fares for supported vehicle types |
| `POST` | `/rides/create` | Rider | Create a ride request |
| `POST` | `/rides/confirm` | Captain | Accept a ride |
| `GET` | `/rides/start-ride?rideId=...&otp=...` | Captain | Validate the OTP and start a ride |
| `POST` | `/rides/end-ride` | Captain | Complete an ongoing ride |

The Socket.IO server is attached to the same HTTP server. The frontend and backend use events including `join`, `update-location-captain`, `new-ride`, `ride-confirmed`, `ride-started`, and `ride-ended`.

## Current scope and notes

- The payment button and payment-related ride fields are present in the interface/model, but a payment provider or payment-processing flow is not implemented.
- Google Maps and geolocation depend on valid API configuration, API restrictions/billing as required by Google, network access, and browser location permission.
- Captain location updates are sourced from browser geolocation; users must grant permission for accurate location.
- Fare rates are currently defined in the backend ride service and are estimates, not a live pricing or payment quote.
- The backend package's `test` script is a placeholder and exits with an error; automated backend tests are not configured yet.

## Available scripts

### Frontend (`uber-fullstack-clone/frontend`)

| Command | Description |
| --- | --- |
| `npm run dev` | Start the Vite development server |
| `npm run build` | Build the frontend for production |
| `npm run preview` | Preview the production build locally |
| `npm run lint` | Run ESLint |

### Backend (`uber-fullstack-clone/Backend`)

| Command | Description |
| --- | --- |
| `node server.js` | Start the Express and Socket.IO server |
