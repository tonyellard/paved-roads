# AWS Lambda Python 3.11 base (ARM64)
FROM public.ecr.aws/lambda/python:3.11-arm64

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt -t /var/task
COPY app.py /var/task/

CMD ["app.handler"]
