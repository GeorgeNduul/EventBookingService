# EventBookingService — Campus Event Management REST API

A Jakarta EE / JAX-RS REST API for creating, discovering, and managing campus events, paired with a Java Swing desktop client. Events are persisted in Azure Cosmos DB, and event details are enriched with live weather and nearby-landmark data from external APIs.

## Features

- **Create events** — publish a new event with title, type, date, cost, location, and coordinates.
- **Search** — case-insensitive partial-match search across title, type, and location.
- **Promoted events** — list events flagged for high-visibility promotion.
- **Registration** — register a student's interest in an event (deduplicated).
- **Attendance tracking** — mark a student as having attended an event.
- **Ratings** — submit a rating/comment for an event, timestamped server-side.
- **Enriched event details** — fetches live weather for the event's city (OpenWeatherMap) with a dress/prep recommendation, plus up to 3 nearby points of interest (GeoNames) based on the event's latitude/longitude.
- **Delete events** — remove an event by ID.
- **Desktop GUI client** — a Swing application (`EventClientGUI`) for exercising the API without a separate frontend.
- **Dockerized** — multi-stage build that compiles the WAR with Maven and deploys it to Tomcat.

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 17 |
| API framework | Jakarta EE / JAX-RS (Jersey 2.42) |
| Build | Maven (packaged as a `.war`) |
| Database | Azure Cosmos DB (SQL API) |
| JSON | Jackson, org.json |
| Application server | Apache Tomcat 9 |
| Desktop client | Java Swing (`java.net.http.HttpClient`) |
| Containerization | Docker (multi-stage build) |
| External APIs | OpenWeatherMap (weather), GeoNames (nearby landmarks) |

## Project Structure

```
EventBookingService/
├── Dockerfile                     # Multi-stage build: Maven build -> Tomcat runtime
├── pom.xml                        # Maven project & dependencies
├── student_meetup.json            # Sample event data
└── src/main/
    ├── java/
    │   ├── EventClientGUI.java                          # Swing desktop client
    │   ├── myRESTws/
    │   │   ├── EventResource.java                       # JAX-RS endpoints (/api/events)
    │   │   └── ExternalApiService.java                  # OpenWeatherMap + GeoNames integration
    │   └── com/mycompany/cwkmaven/
    │       └── JakartaRestConfiguration.java             # JAX-RS application path (/api)
    ├── resources/META-INF/persistence.xml
    └── webapp/                                           # Static assets, web.xml, beans.xml
```

## API Reference

Base path: `/api/events`

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/events` | Create a new event. Auto-fills `id`, `attendees`, `attendance_list`, `ratings`, `is_advertised` if not supplied. |
| `GET` | `/api/events/search?query=` | Search events by title, type, or location (case-insensitive, partial match). Returns all events if `query` is empty. |
| `GET` | `/api/events/promoted` | List events where `is_advertised = true`. |
| `POST` | `/api/events/{event_id}/register` | Register a student (`student_id` in body) as an attendee. |
| `POST` | `/api/events/{event_id}/attendance` | Mark a student (`student_id` in body) as having attended. |
| `POST` | `/api/events/{event_id}/rate` | Submit a rating for an event (rating data in body; server stamps `rating_date`). |
| `GET` | `/api/events/{event_id}/details` | Get an event enriched with live weather + a dress recommendation, and up to 3 nearby landmarks (if `lat`/`lng` are set). |
| `DELETE` | `/api/events/{event_id}` | Delete an event by ID. |

## Getting Started

### Prerequisites

- Java 17
- Maven 3.8+
- An Azure Cosmos DB account (SQL API) with a database/container to store events
- API keys for [OpenWeatherMap](https://openweathermap.org/api) and [GeoNames](https://www.geonames.org/export/web-services.html)
- Docker (optional, for containerized deployment)

### Configuration

`EventResource.java` and `ExternalApiService.java` currently hold the Cosmos DB endpoint/key and the weather/GeoNames credentials as constants. Before running this yourself, replace them with your own credentials — ideally sourced from environment variables or a secrets manager rather than hardcoded in source, since this repo is public.

### Build and run locally

```bash
mvn clean package
```

Deploy the resulting `target/CWKMAVEN-1.0-SNAPSHOT.war` to a local Tomcat 9 instance, then the API is reachable at:

```
http://localhost:8080/CWKMAVEN-1.0-SNAPSHOT/api/events
```

### Run with Docker

```bash
docker build -t event-booking-service .
docker run -p 9090:9090 event-booking-service
```

The Dockerfile builds the WAR with Maven, deploys it to Tomcat as `ROOT.war`, and exposes port `9090`, so once running the API is reachable at:

```
http://localhost:9090/api/events
```

### Desktop client

`EventClientGUI.java` is a standalone Swing app that talks to the REST API. Update the `BASE_URL` constant at the top of the file to match wherever you deployed the API, then compile and run it with the same dependencies as the server (Jersey client libs aren't required — it uses `java.net.http.HttpClient`).

### Sample data

`student_meetup.json` contains 10 sample campus events (sports, concerts, workshops, trips, etc.) that can be used to seed the database or as example payloads for the `POST /api/events` endpoint.

## Notes

- Search queries are parameterized via `SqlQuerySpec` to prevent SQL/NoSQL injection.
- State-changing operations (register, attendance, rate) use a read-modify-replace pattern against Cosmos DB rather than partial updates.
