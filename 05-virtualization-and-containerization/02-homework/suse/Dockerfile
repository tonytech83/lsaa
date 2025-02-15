FROM opensuse/leap:15.6

# set the working directory
WORKDIR /srv/www/htdocs

# install Apache
RUN zypper refresh && \
    zypper install -y apache2 && \
    zypper clean --all

# copy index.html to workdir
COPY index.html /srv/www/htdocs/index.html

EXPOSE 80

# Start Apache server
CMD ["/usr/sbin/httpd", "-D", "FOREGROUND"]
