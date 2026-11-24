# DropOff Security Assignment

**Hosted application:** [https://cisc472-assignment7.onrender.com/](https://cisc472-assignment7.onrender.com/)

# Directory Traversal

**Location:** `app.py`, `upload()` and `download()`

**Trigger:** Providing a filename containing path components such as `..` or an absolute path.

*Example:* A logged-in user sends an upload whose multipart filename is `../../private/grader.txt`. The `upload()` function uses `file.filename` as the stored name and writes to `UPLOADS / stored`, allowing the file to be created outside `data/uploads/`. An unauthenticated user can also request `/download?name=../private/grader.txt` or an absolute-looking path. The `download()` function joins `data/uploads/` with the supplied name and may serve `data/private/grader.txt`, which is created at startup and contains "TA only" notes and the release code `DROP-5521`.

**The Fix**:
In `upload()`, generate a server-controlled stored filename, such as a random UUID plus a safe extension. Do not pass `file.filename` directly into a filesystem path. In `download()`, resolve the requested path and verify that it is a regular file contained by `UPLOADS.resolve()`. Reject absolute paths and paths outside the uploads directory. The response should return 404 for invalid or missing files. Authentication and ownership checks should also ensure that a user can download only files associated with that user's account.

**Where an uploaded file lands on disk**

The browser submits the selected file to `POST /upload` in `app.py`. The `upload()` route gets the filename and file bytes, uses the filename as the `stored_name`, and writes the bytes to `UPLOADS / stored_name`. `UPLOADS` is `data/uploads/` by default, so a normal upload named `lab2.pdf` is stored at `data/uploads/lab2.pdf`. The server also inserts the user's ID, stored name, and original name into the `files` table in `data/dropoff.db`.

**How a download request becomes a path**

The browser sends `GET /download?name=lab2.pdf`. The `download()` route in `app.py` takes the `name` value and joins it to `UPLOADS` using `os.path.join(str(UPLOADS), name)`. This produces a path such as `data/uploads/lab2.pdf`, which is passed to `FileResponse`. Because the supplied name is not checked against the uploads directory, a name containing `../` can leave `data/uploads/` and point to another file.

**What a signed-in user can list versus what is only a file on disk**

The home page calls `current_user()` to obtain the signed-in user's database row. It then queries the `files` table for rows whose `user_id` matches that user and passes those rows to `templates/home.html`. The signed-in user is therefore shown only database-registered files belonging to that account. A file that exists on disk but is not in the user's `files` rows does not appear in the list, but the current `/download` route does not verify database ownership and may still serve it when its name is supplied directly.

**What is in `data/` after the first request**

On startup, the application creates `data/uploads/`, `data/private/`, and `data/dropoff.db` if they do not already exist. Startup initialization creates the two demo users, Maya's sample file `data/uploads/lab1-maya.txt`, Maya's row in the `files` table, and `data/private/grader.txt`. The private grader file contains instructor-only notes and the release code `DROP-5521`. `/download` must not expose this file.
