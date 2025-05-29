FROM python:3.9

WORKDIR /app
COPY app.py .
RUN pip install flask requests redis mysql-connector-python
EXPOSE 3002
CMD ["python", "app.py"]
