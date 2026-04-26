FROM python:3.13-slim
LABEL org.opencontainers.image.authors="martial@email.com" \
      org.opencontainers.image.source="https://github.com/Binfl/Projet_Docker1" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.title="simple_api" \
      org.opencontainers.image.description="API de liste des students" 
RUN apt-get update -y && apt-get install -y \
	build-essential \
	python3-dev \
	libsasl2-dev \
	libldap2-dev \
	libssl-dev && \
	rm -rf /var/lib/apt/lists/*
VOLUME /data
WORKDIR /
COPY requirements.txt .
RUN pip3 install -r /requirements.txt
COPY . .
EXPOSE 5000
CMD [ "python3", "./student_age.py" ]
