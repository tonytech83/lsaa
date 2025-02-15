FROM centos/httpd

# set the working directory
WORKDIR /var/www/html

# copy index.html into WORKDIR 
COPY index.html .

# set correct permissions for WORKDIR
RUN chmod -R 755 .

# expose port 80
EXPOSE 80

# start Apache in foreground mode
ENTRYPOINT ["httpd","-D","FOREGROUND"]