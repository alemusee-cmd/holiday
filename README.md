# Holiday Events

Holiday Events is a small internal company portal. Employees can browse upcoming holiday gatherings and register for an event.

## Requirements

- Node.js 18 or later
- npm

## Install

```bash
npm install
```

## Start

```bash
npm start
```

The same command is available as `npm run dev`.

The server uses the `PORT` environment variable and defaults to `3000` when it is not set.

```bash
PORT=3000 npm start
```

## Access the application

Open a browser and go to:

[http://localhost:3000](http://localhost:3000)

## API endpoints

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/health` | Health check. Returns `{ "status": "healthy", "service": "holiday-events" }`. |
| `GET` | `/api/events` | Returns the list of events. |
| `GET` | `/api/events/:id` | Returns a single event. |
| `POST` | `/api/register` | Registers a person for an event. |

### Registration body

```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "eventId": 1
}
```

A successful registration decreases the number of remaining spots for that event. Event availability is kept in memory while the server is running.
