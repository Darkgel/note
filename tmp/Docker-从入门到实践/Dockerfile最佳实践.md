# Dockerfile最佳实践

每个指令会创建一个镜像层，在从镜像中启动容器时，会在底层的基础上创建一个writable layer (the “container layer”)

指引和建议

- Create ephemeral containers：这些容器可以被停止和摧毁，然后rebuilt和替换
- Understand build context：当使用docker build命令时，当前的工作目录（working directory）被称为build context。不管Dockerfile在哪里，所有的working directory下的东西会被发送到Docker daemon作为build context
- Pipe Dockerfile through stdin：常用于一次性构建，不将Dockerfile存储到磁盘中。当使用符号“-”时表示除了Dockerfile，不将其他任何东西发送给Docker daemon作为build context(docker build \[OPTIONS\] -)
- Exclude with .dockerignore: 作用类似.gitignore
- Use multi-stage builds: 使用 multi-stage build可以减小最终镜像的大小
- Don’t install unnecessary packages：
- Decouple applications：Each container should have only one concern
- Minimize the number of layers：
  - Only the instructions RUN, COPY, ADD create layers.Other instructions create temporary intermediate images, and do not increase the size of the build.
  - Where possible, use multi-stage builds, and only copy the artifacts you need into the final image. This allows you to include tools and debug information in your intermediate build stages without increasing the size of the final image.
- Sort multi-line arguments
- Leverage build cache

## Dockerfile instructions

- FROM
  - Whenever possible, use current official images as the basis for your images. （推荐Alpine image）
- LABEL
  - add labels to your image to help organize images by project, record licensing information, to aid in automation, or for other reasons.
- RUN
  - Split long or complex RUN statements on multiple lines separated with backslashes to make your Dockerfile more readable, understandable, and maintainable.
  - APT-GET
    - Avoid RUN apt-get upgrade and dist-upgrade, as many of the “essential” packages from the parent images cannot upgrade inside an unprivileged container.
    - Always combine RUN apt-get update with apt-get install in the same RUN statement. 
  - USING PIPES
- CMD
  - CMD should rarely be used in the manner of CMD ["param", "param"] in conjunction with ENTRYPOINT, unless you and your expected users are already quite familiar with how ENTRYPOINT works.
  - CMD should almost always be used in the form of CMD ["executable", "param1", "param2"…].
- EXPOSE
- ENV
  - ENV can also be used to set commonly used version numbers so that version bumps are easier to maintain
- ADD or COPY
  - Although ADD and COPY are functionally similar, generally speaking, COPY is preferred. 
  - If you have multiple Dockerfile steps that use different files from your context, COPY them individually, rather than all at once. This ensures that each step’s build cache is only invalidated (forcing the step to be re-run) if the specifically required files change.
- ENTRYPOINT
- VOLUME
- USER
  - If a service can run without privileges, use USER to change to a non-root user. 
- WORKDIR
  - For clarity and reliability, you should always use absolute paths for your WORKDIR
- ONBUILD