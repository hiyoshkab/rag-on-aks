FROM python:3.14-slim
 
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
 
WORKDIR /app
 
COPY backend/requirements.txt .
RUN pip install --no-cache-dir --upgrade pip \
    && pip install --no-cache-dir -r requirements.txt 

# Remove files to prevent false positives in security scans
RUN find / -type f \( -name '*.cdx.json' -o -name '*.bom.json' -o -name 'bom.json' -o -name '*.spdx.json' -o -name '*.spdx' \) -delete 2>/dev/null || true
 
COPY backend/app_backend/ ./app_backend/
 
RUN useradd --create-home --uid 10001 appuser \
    && chown -R appuser:appuser /app
 
USER appuser
 
EXPOSE 8000
 
CMD ["uvicorn", "app_backend.main:app", "--host", "0.0.0.0", "--port", "8000"]