# Visualizer

## Quick Start

1. **Build the Project**

    ```sh
    mvn clean install
    ```

2. **Visit Swagger for API Documentation**

    [http://localhost:8080/swagger-ui.html](http://localhost:8080/swagger-ui.html)

---

## REST APIs Design

### 1. File Upload Endpoint

- **URL:** `POST /api/v1/maps/uploads`
- **Auth:** Required (User must be logged in)
- **Request:** `multipart/form-data` containing the file.
    - `file`: The actual file blob.
    - *(Optional)* `name`: A user-friendly name for this map/dataset.
    - *(Optional)* `description`: A description.

**Processing:**
- Validate file type and size.
- Save the file to temporary storage or cloud storage (e.g., S3).
- Create an UploadedFile record in the database with `status=PENDING`.
- Trigger an asynchronous background task to process the file, passing the UploadedFile ID and storage path.

**Response:**  
`202 Accepted` — Indicates the request is accepted for processing.

**Body (JSON):**
```json
{
  "upload_id": "unique_upload_file_id_123",
  "status": "PENDING",
  "message": "File uploaded successfully and queued for processing."
}
```

---

### 2. Upload Status Endpoint

- **URL:** `GET /api/v1/maps/uploads/{upload_id}/status`
- **Auth:** Required (User must own the upload)

**Processing:**
- Fetch the UploadedFile record by `upload_id`.
- Check user ownership.

**Response:**  
`200 OK`
```json
{
  "upload_id": "unique_upload_file_id_123",
  "status": "PROCESSING", // or PENDING, COMPLETED, FAILED
  "detected_geo_level": "STATE", // Available after partial/full processing
  "detected_metrics": ["population", "literacy"], // Available after partial/full processing
  "error_message": null // or "Error details..." if status is FAILED
}
```
- `404 Not Found`: If `upload_id` doesn't exist or user doesn't have access.

---

### 3. Map Data Retrieval Endpoint

- **URL:** `GET /api/v1/maps/uploads/{upload_id}/data`
- **Auth:** Required (User must own the upload)
- **Query Parameters:**
    - `metric`: String (e.g., population). **Required.**
    - *(Optional)* `geo_level`: String (e.g., STATE, DISTRICT).

**Processing:**
- Fetch the UploadedFile record. Ensure status is `COMPLETED`. Check ownership.
- Query the GeoDataPoint table for records linked to this `upload_id`.
- Filter by `geo_level` if provided.
- Filter out features where `is_matched` is false (or handle them differently if needed).
- Construct a GeoJSON FeatureCollection.

**Response:**  
`200 OK`  
**Body: GeoJSON FeatureCollection**
```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": { /* GeoJSON Polygon or MultiPolygon */ },
      "properties": {
        "name": "California",
        "metric_name": "population",
        "value": 39500000
        // optional other properties like 'literacy': 92.5
      }
    },
    {
      "type": "Feature",
      "geometry": { /* GeoJSON Polygon or MultiPolygon */ },
      "properties": {
        "name": "Texas",
        "metric_name": "population",
        "value": 29100000
      }
    }
    // ... more features
  ]
}
```
- `400 Bad Request`: If `metric` parameter is missing/invalid, or if the requested metric wasn't found.
- `404 Not Found`: If `upload_id` doesn't exist or user doesn't have access.
- `409 Conflict`: If the upload processing is not yet `COMPLETED`.

---

### 4. (Optional) List User Uploads Endpoint

- **URL:** `GET /api/v1/maps/uploads`
- **Auth:** Required

**Processing:** Fetch all UploadedFile records for the current user.

**Response:**  
`200 OK` with a list of upload summaries (ID, name, status, timestamp, etc.).

---

## API Endpoints Summary

| #   | Method | Endpoint                                 | Auth | Description                                                   | Response (Status)                            |
|----:|--------|------------------------------------------|------|---------------------------------------------------------------|----------------------------------------------|
| 1   | POST   | `/api/v1/maps/uploads`                   | ✅    | Upload a file (CSV, Excel, PDF, etc.) for processing          | `202 Accepted` with `upload_id`, status, message |
| 2   | GET    | `/api/v1/maps/uploads/{upload_id}/status`| ✅    | Get the status of an uploaded file                            | `200 OK` with status, detected metrics, errors   |
| 3   | GET    | `/api/v1/maps/uploads/{upload_id}/data`  | ✅    | Fetch processed GeoJSON data with a specific metric and level | `200 OK` with FeatureCollection GeoJSON          |
| 4   | GET    | `/api/v1/maps/uploads`                   | ✅    | (Optional) List all uploads by the current user               | `200 OK` with list of upload summaries           |

| Endpoint                                   | Required Params / Query                                         | Notes                                                         |
|---------------------------------------------|-----------------------------------------------------------------|---------------------------------------------------------------|
| `/api/v1/maps/uploads`                      | `file (form-data)`<br>Optional: `name`, `description`           | Triggers background processing, stores data with status `PENDING` |
| `/api/v1/maps/uploads/{upload_id}/status`   | Path: `upload_id`                                               | Returns status: `PENDING`, `PROCESSING`, `COMPLETED`, `FAILED`    |
| `/api/v1/maps/uploads/{upload_id}/data`     | Path: `upload_id`<br>Query: `metric` (required), `geo_level` (optional) | Returns valid GeoJSON only if status is `COMPLETED`               |
| `/api/v1/maps/uploads`                      | None                                                            | Used to show upload history, possibly for a dashboard             |

---

## Extended Endpoints

| #   | Method | Endpoint                                         | Description                                                                          |
|----:|--------|--------------------------------------------------|--------------------------------------------------------------------------------------|
| 1   | POST   | `/api/v1/uploads`                                | Upload a CSV/Excel/PDF file. Returns `uploadId`                                      |
| 2   | GET    | `/api/v1/uploads/{uploadId}/status`              | Get parsing status: `PENDING`, `PROCESSING`, `COMPLETED`, `FAILED`                   |
| 3   | GET    | `/api/v1/uploads/{uploadId}/summary`             | Returns inferred metadata: column names, types, geo info, time-series detection, metrics |
| 4   | GET    | `/api/v1/uploads/{uploadId}/data`                | Returns cleaned JSON data (tabular)                                                  |
| 5   | GET    | `/api/v1/uploads/{uploadId}/visualization/map`   | Returns GeoJSON FeatureCollection for mapping (if geo data is detected)              |
| 6   | GET    | `/api/v1/uploads/{uploadId}/visualization/chart` | Returns aggregated data suitable for bar/line/pie chart rendering                    |
| 7   | GET    | `/api/v1/uploads`                                | List past uploads with basic metadata for the logged-in user                         |
