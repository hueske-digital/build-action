# The image the tests build: the architecture it was built for, and a
# build argument.
FROM busybox:1.37
ARG GREETING=unset
RUN echo "$GREETING $(uname -m)" > /built
CMD ["cat", "/built"]
