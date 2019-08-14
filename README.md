# testapp-travis-ci-deploy

```
Travis/CI:  https://travis-ci.org/

Log in with GITHUB ID.

Travis CI is a hosted continuous integration service used to build and test software projects hosted at GitHub.

Open source projects may be tested at no charge via travis-ci.org. Private projects may be tested at travis-ci.com on a fee basis. TravisPro provides custom deployments of a proprietary version on the customer's own hardware. 

Continuous integration (CI) is a DevOps practice in which team members regularly commit their code changes to the version control repository, after which automated builds and tests are run.  Continuous delivery (CD) is a series of practices where code changes are automatically built, tested and deployed to production.

The advent of Agile development means developers can deliver software products tailored to meet customer design specifications and requirements. In an Agile development environment, smaller groups in a team often work on different aspects of a larger project and commit regularly. The frequent addition of smaller chunks of code to the codebase through version control ensures that previous work is recoverable. However, this frequent addition of code to the codebase can break the existing codebase without proper communication and monitoring. To minimize this manual scrutiny and redundant communication, we need to invest in CI/CD processes. In  this article, we will walk through an end-to-end workflow of automating a CI/CD process for a Flask application via Travis CI and Heroku.




   Build, Test & Deploy phases.


Disable Firewall on Ubuntu 18.04
$ sudo ufw status
$ sudo ufw disable
$ sudo systemctl disable ufw
$ sudo systemctl stop ufw




   1: Provision an instance with Terraform.

$ git clone https://github.com/rgdevops123/terraform-aws-ec2-instance-ubuntu
$ cd terraform-aws-ec2-instance
   Follow README.md instructions.


         Part 1: The Web Application.

   Create a Test Flask Application:
      $ git clone https://github.com/rgdevops123/testapp-travis-ci-deploy.git



$ vim requirements.txt
+++
flask
xmlrunner
+++

$ sudo apt -y install python3-pip
$ sudo pip3 install -r requirements.txt


$ vim app.py
+++
#!/usr/bin/python3

from flask import Flask


app = Flask(__name__)


@app.route('/')
@app.route('/hello/')
def hello_world():
    return 'Hello World!\n'


@app.route('/hello/<username>')  # Dynamic route.
def hello_user(username):
    return 'Why Hello %s!\n' % username


if __name__ == '__main__':
    app.run(host='0.0.0.0')      # Open for everyone.
+++


$ chmod +x app.py

$ ./app.py &



   Test the server:
$ curl -i localhost:5000/
$ curl -i localhost:5000/hello/
$ curl -i localhost:5000/hello/John



         The Unit Tests

   Write some tests to test these routes.

$ vim test.py
+++
#!/usr/bin/python3
import unittest
import app


class TestHello(unittest.TestCase):

    def setUp(self):
        app.app.testing = True
        self.app = app.app.test_client()

    def test_hello(self):
        rv = self.app.get('/')
        self.assertEqual(rv.status, '200 OK')
        self.assertEqual(rv.data, b'Hello World!\n')

    def test_hello_hello(self):
        rv = self.app.get('/hello/')
        self.assertEqual(rv.status, '200 OK')
        self.assertEqual(rv.data, b'Hello World!\n')

    def test_hello_name(self):
        name = 'John'
        rv = self.app.get(f'/hello/{name}')
        self.assertEqual(rv.status, '200 OK')
        self.assertIn(bytearray(f"{name}", 'utf-8'), rv.data)


if __name__ == '__main__':
    import xmlrunner
    unittest.main(testRunner=xmlrunner.XMLTestRunner(output='test-reports'))
    unittest.main()
+++



$ chmod +x test.py


   Run the Tests:

$ ./test.py


   Check the reports in test-reports.
$ cat test-reports/TEST-TestHello-<TIME_AND_DATE_OF_TEST>.xml



         Part 2: Docker.

   Create a Dockerfile, Docker Image and Container:

    Install Docker:
$ sudo apt -y install docker.io
$ sudo systemctl start docker
$ sudo systemctl enable docker

$ vim Dockerfile
+++
# Use a Python 3.6 Base Image.
FROM python:3.6

# Set Maintainer.
LABEL maintainer "rgdevops123@gmail.com"

# Set Environment variable.
ENV TESTAPP_VERSION 0.0.1

# Copy Application files.
COPY app.py requirements.txt test.py /

# Install Dependencies.
RUN pip install -r requirements.txt

# Set a Health Check.
HEALTHCHECK --interval=5s \
            --timeout=5s \
            CMD curl -f http://127.0.0.1:5000 || exit 1

# tell docker what port to expose
EXPOSE 5000

# Specify the command to run.
ENTRYPOINT ["python","app.py"]
+++


   Build the Image.
$ sudo docker build . -t rgdevops123/testapp-travis-ci-deploy

   Confirm:
$ sudo docker images


   Create a Container from the image.
$ sudo docker run -d --rm --name testapp-travis-ci-deploy -p 5000:5000 rgdevops123/testapp-travis-ci-deploy

   Confirm:
$ sudo docker ps
      AND
   GOTO>>> http://127.0.0.1:5000/





         Part 3: The Travis CI Pipeline.
https://travis-ci.org/

Log in with GITHUB ID.

After signing in, you will find a list of all your public GitHub repositories. To enable Travis CI, track your repository, and build your project for each push, flip the switch next to your project repository.

Enable tracking for your GitHub repo on Travis CI

After activating Travis CI to track your project repository, add a .travis.yml file to the root of your project directory and add the following lines to it:




$ vim .travis.yml
+++
sudo: required
services:
  - docker
env:
  global:
    - IMAGE_NAME=rgdevops123/testapp-travis-ci-deploy
language: python
python:
  - "3.6"
install:
  - pip install -r requirements.txt
script:
  # Unit Test
  - python test.py
after_success:
  - version="$(awk '$2 == "TESTAPP_VERSION" { print $3; exit }' Dockerfile)"
  - docker pull "$IMAGE_NAME" || true
  - docker build --pull --cache-from "$IMAGE_NAME" --tag "$IMAGE_NAME" .
  - docker login -u "$DOCKER_USER" -p "$DOCKER_PASS"
  - docker tag "$IMAGE_NAME" "${IMAGE_NAME}:latest"
  - docker tag "$IMAGE_NAME" "${IMAGE_NAME}:${version}"
  - docker push "${IMAGE_NAME}:latest" && docker push "${IMAGE_NAME}:${version}"
+++


Under the Settings view of your repository on Travis CI, set the environment variables used in the script as shown:

DOCKER_USER = rgdevops123
DOCKER_PASS = <PASSWORD>
```
