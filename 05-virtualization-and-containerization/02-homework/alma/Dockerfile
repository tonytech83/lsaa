FROM centos/httpd

# Set the working directory
WORKDIR /var/www/html

# Copy index.html into WORKDIR 
COPY index.html .

# Set correct permissions for WORKDIR
RUN chmod -R 755 .

# Expose port 80
EXPOSE 80

# Start Apache in foreground mode
ENTRYPOINT ["httpd","-D","FOREGROUND"]