FROM java:8-jre
MAINTAINER <ssab work_wjj@163.com>
LABEL Description="使用mycat做mysql数据库的读写分离"
ENV mycat-version Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz
USER root
COPY ./Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz /
RUN tar -zxf /Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz
ENV MYCAT_HOME=/mycat
ENV PATH=$PATH:$MYCAT_HOME/bin
WORKDIR $MYCAT_HOME/bin
RUN chmod u+x ./mycat
EXPOSE 8066 9066
CMD ["./mycat","console"]
