# Build a FastAPI code runner endpoint for tool calls

This guide shows how to deploy a minimal FastAPI service that can execute short Python snippets sent from an agent. The service checks a bearer token, writes the submitted code to a temporary file, runs it with `python`, and returns the stdout/stderr/exit code as JSON. The API is intentionally constrained to Python for safety and predictability.

## Prerequisites

- Python 3.9+
- `fastapi` and `uvicorn`
- An environment variable `RUNNER_API_KEY` that you will pass as a bearer token from your client or agent tool

Install dependencies:

```bash
pip install fastapi uvicorn
```

## FastAPI server

Save the following as `app.py`:

```python
from fastapi import FastAPI, Header, HTTPException
import subprocess, tempfile, textwrap, os

API_KEY = os.environ["RUNNER_API_KEY"]
app = FastAPI()

@app.post("/run")
def run(payload: dict, authorization: str = Header("")):
    if authorization != f"Bearer {API_KEY}":
        raise HTTPException(401, "bad token")
    lang = payload.get("language", "python")
    code = payload["code"]
    if lang != "python":
        raise HTTPException(400, "only python")
    with tempfile.NamedTemporaryFile("w", suffix=".py", delete=False) as f:
        f.write(textwrap.dedent(code))
        path = f.name
    try:
        out = subprocess.run(
            ["python", path],
            capture_output=True,
            text=True,
            timeout=10  # seconds
        )
        return {
            "stdout": out.stdout,
            "stderr": out.stderr,
            "returncode": out.returncode
        }
    finally:
        os.remove(path)
```

Start the server locally:

```bash
export RUNNER_API_KEY="your-secret"
uvicorn app:app --host 0.0.0.0 --port 8000
```

## OpenAPI description

Expose the endpoint to an agent client with this OpenAPI description (for example, when registering the tool in Chat Completions):

```yaml
openapi: 3.1.0
info:
  title: Code Runner
  version: 1.0.0
servers:
  - url: https://your-runner.example.com
paths:
  /run:
    post:
      operationId: runCode
      summary: Execute code in a sandbox
      security:
        - ApiKeyAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - code
              properties:
                language:
                  type: string
                  enum:
                    - python
                code:
                  type: string
      responses:
        "200":
          description: Execution result
          content:
            application/json:
              schema:
                type: object
                properties:
                  stdout:
                    type: string
                  stderr:
                    type: string
                  returncode:
                    type: integer
components:
  securitySchemes:
    ApiKeyAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

## Sending a request

Use `curl` to run a short snippet and inspect the output:

```bash
curl -X POST "http://localhost:8000/run" \
  -H "Authorization: Bearer $RUNNER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "language": "python",
    "code": "print(\"Hello from the runner!\")"
  }'
```

You should receive a JSON response similar to:

```json
{
  "stdout": "Hello from the runner!\n",
  "stderr": "",
  "returncode": 0
}
```

## Safety and customization tips

- Restrict the runner to a dedicated sandbox with limited filesystem and network access; the example only enforces a language check and a 10-second timeout.
- Log requests and responses carefully, redacting secrets, to make debugging easier without leaking code.
- Adjust the timeout, allowlists, or interpreter command if you need additional languages.
- Keep `RUNNER_API_KEY` secret and rotate it periodically; the endpoint denies requests without a matching bearer token.
```
