FROM debian:bullseye

# Collection of packages that may or may not be actually required by the project (copied from the project I'm mimicking)
ADD /apt-package-list /
RUN apt-get update && apt-get install -y $(grep -vE "^\s*#" /apt-package-list  | tr "\n" " ")

ARG PYTHON3_VERSION=3.7.3
RUN curl --silent --show-error --retry 5 https://bootstrap.pypa.io/get-pip.py | /usr/bin/python3 - 'pip==21.3.1'

# The root of the app in the project
RUN mkdir /opt/sample

# Setting up SSH in the container
RUN mkdir /var/run/sshd
RUN echo 'root:root' | chpasswd
RUN sed -i 's/PermitRootLogin without-password/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
# SSH login fix. Otherwise user is kicked off after login
RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd
ENV NOTVISIBLE "in users profile"
RUN echo "export VISIBLE=now" >> /etc/profile

# only allow localhost ssh connection
RUN echo "sshd: ALL" >> /etc/hosts.deny
RUN echo "sshd: 192.168.64.1" >> /etc/hosts.allow

ADD /requirements /
RUN /usr/local/bin/pip3 install -r /requirements

WORKDIR /opt/sample

EXPOSE 8000
EXPOSE 22

ADD ./startup.sh /
CMD /usr/local/bin/gunicorn sammple.wsgi:application -b 0.0.0.0 -w 16 -t 90 --limit-request-line 0 --max-requests 200000 --max-requests-jitter 50
