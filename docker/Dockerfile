FROM eclipse-temurin:17-jre

# 设置时区
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 创建工作目录
WORKDIR /app

# 创建文件上传目录
RUN mkdir -p /home/ruoyi/uploadPath

# 复制 jar 包
COPY ruoyi-admin.jar /app/ruoyi-admin.jar

# 暴露端口
EXPOSE 8080

# 启动应用
ENTRYPOINT ["java", "-jar", "-Duser.timezone=Asia/Shanghai", "/app/ruoyi-admin.jar"]
