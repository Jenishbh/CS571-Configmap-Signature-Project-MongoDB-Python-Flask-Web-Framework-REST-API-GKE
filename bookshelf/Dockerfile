FROM python:3.8-slim-buster


WORKDIR /app

RUN pip3 install flask
RUN pip3 install flask_pymongo

COPY . /app
ENV PORT 5000
EXPOSE 5000
ENTRYPOINT [ "python3" ]
CMD [ "bookshelf.py" ]