FROM python:latest

WORKDIR /usr/src

RUN apt-get update && \
    apt-get install -y \
    vim \
    git

ADD requirements.txt .
RUN pip install -r requirements.txt

ADD . .

EXPOSE 8000

CMD ["fastapi", "run", "app.py"]
