# --- Build Stage ---
FROM python:3.11-slim AS base

WORKDIR /app

# Install dependencies first (layer caching)
COPY requirements.txt requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Expose port
EXPOSE 80

# Run with gunicorn for production
CMD ["gunicorn", "app:app", "-b", "0.0.0.0:80", "--log-file", "-", "--access-logfile", "-", "--workers", "4", "--keep-alive", "0"]
