FROM python:3.11

WORKDIR /app

COPY requirements.txt .

RUN pip install -r requirements.txt

COPY . /app

EXPOSE 5000

RUN chmod +x script.sh

RUN bash script.sh

CMD ["python", "webapp/run.py"]
