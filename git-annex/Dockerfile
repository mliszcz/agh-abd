
FROM centos:6

RUN yum -y install \
    epel-release \
    sudo \
    vim

RUN yum -y install \
    git \
    git-annex \
    rsync \
    openssh-server

ENV RSA1_KEY=/etc/ssh/ssh_host_key \
    RSA_KEY=/etc/ssh/ssh_host_rsa_key \
    DSA_KEY=/etc/ssh/ssh_host_dsa_key

RUN /usr/bin/ssh-keygen -q -t rsa1 -f $RSA1_KEY -C '' -N '' && \
    /usr/bin/ssh-keygen -q -t rsa -f $RSA_KEY -C '' -N '' && \
    /usr/bin/ssh-keygen -q -t dsa -f $DSA_KEY -C '' -N '' && \
    chmod 600 $RSA1_KEY $RSA_KEY $DSA_KEY && \
    chmod 644 $RSA1_KEY.pub $RSA_KEY.pub $DSA_KEY.pub

RUN sed -i -E 's|^.?(UseDNS).*$|\1 no|g' /etc/ssh/sshd_config && \
    sed -i -E 's|^.?(GSSAPIAuthentication).*$|\1 no|g' /etc/ssh/sshd_config

RUN echo '%wheel ALL=(ALL) ALL' >> /etc/sudoers && \
    useradd -m -s /bin/bash -G wheel git && \
    echo 'git:git' | chpasswd

USER git

RUN git config --global user.email "git@agh.edu.pl" && \
    git config --global user.name "Git Git"

USER root

CMD /usr/sbin/sshd -De
