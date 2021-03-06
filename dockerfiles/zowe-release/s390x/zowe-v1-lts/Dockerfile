FROM ompzowe/zowecicd-node12-java8:s390x

# allow to customize which Zowe build to build from
# There are 3 possible values:
# 1. ZOWE_BUILD=: empty, which will try to find the local zowe.pax in same directory. The build has to be named as zowe.pax.
# 2. ZOWE_BUILD=latest: will download the most recent official release and use it as base build
# 3. ZOWE_BUILD=<url>: will download Zowe build from this url and use it as base build
ARG ZOWE_BUILD=

EXPOSE 7553/tcp
EXPOSE 7554/tcp
EXPOSE 8544/tcp

#ENV ZOWE_ZOSMF_HOST='zosmf.host.com'
#ENV ZOWE_ZSS_HOST='zss.host.com'
#ENV ZWED_agent_host='zwed.host.com'
ENV ZOWE_EXPLORER_HOST='localhost'
ENV ZOWE_IP_ADDRESS=0.0.0.0

ENV ZOWE_ZOSMF_PORT='443'
#ENV ZOWE_ZSS_SERVER_PORT='00000'
ENV ZWED_agent_http_port='8542'

ENV LAUNCH_COMPONENT_GROUPS=DESKTOP,GATEWAY

RUN mkdir /root/zowe/utils && \
    mkdir /root/zowe/samples
COPY utils/ /root/zowe/utils/
COPY start.sh /root/zowe/samples/

RUN cd /root/zowe/utils/autoconv && \
    npm install && \
    npm run build && \
    rm -rf node_modules src samples *.js *.json .editorconfig *ignore *.md && \
    ln -s ./autoconv/bin/autoconv ../autoconv.sh

# copy run.sh and optional zowe.pax
COPY run.sh zowe.pax* /root/zowe/
# download zowe build if specified
RUN /root/zowe/utils/download-zowe-build.sh "${ZOWE_BUILD}"

# 1. Convert content of pax files from ebcdic to ASCII
# 2. replace or remove some zos specific parameters
# 3. Install zowe, then remove installer files
RUN cd /root/zowe/zowe-install && \
    tar -xvf ../zowe.pax --strip 1 && \
    find . -type f -iregex '.*\.\(rexx\|js\|sh\|json\|jcl\|yaml\|clist\|env\)$' -exec sh -c "conv '{}' | sponge '{}'" \; && \
    find . -type f -name '*.sh' -exec sh -c "sed -i 's/-Xquickstart//' {}" \; && \
    find . -type f -name '*.sh' -exec sh -c "sed -i 's/-ppx/-pp/' {}" \; && \
    find . -type f -name '*.sh' -exec sh -c "sed -i 's/iconv -f IBM-1047 -t IBM-850/cat/' {}" \; && \
    find . -type f -name '*.sh' -exec sh -c "chmod +x {}" \; && \
    echo "exit 0" > scripts/opercmd && \
    sed -i 's/-px //' scripts/zowe-install-zlux.sh && \
    sed -i 's/java version/openjdk version/' scripts/utils/validate-java.sh && \
    \
    \
    cd /root/zowe/zowe-install/files && \
    for f in *.pax; do \
        echo "Processing $f file." && \
        rm -rf tmp && \
        mkdir tmp && \
        cd tmp && \
        pax -rf ../$f && \
        rm ../$f && \
        find . -type f -iregex '.*\.\(rexx\|js\|sh\|json\|jcl\|yaml\|clist\|html\|template\|css\|svg\|map\|gz\|env\)$' -exec sh -c "conv '{}' | sponge '{}'" \; && \
        find . -type f -name '*.sh' -exec sh -c "sed -i 's/-Xquickstart//' {}" \; && \
        pax -wf ../$f . && \
        cd .. ;  \
    done && \
    cd zlux && \
    for f in *.pax; do \
        echo "Processing $f file." &&\
        rm -rf tmp &&\
        mkdir tmp &&\
        cd tmp &&\
        pax -rf ../$f &&\
        rm ../$f &&\
        find . -type f -iregex '.*\.\(sh\)$' -exec sh -c "conv '{}' | sponge '{}'" \; && \
        find . -type f -name '*.sh' -exec sh -c "sed -i 's/-Xquickstart//' {}" \; &&\
        pax -wf ../$f . &&\
        cd .. ; \
    done && \
    \
    \
    /root/zowe/zowe-install/install/zowe-install.sh -i /root/zowe/install -l /root/zowe/logs && \
    bash -c "/root/zowe/install/bin/zowe-configure-instance.sh -c /root/zowe/instance" && \
    \
    \
    cd /root/zowe/install/components/explorer-ui-server && \
    rm -rf node_modules && \
    npm install --only=prod --ignore-scripts && \
    rm -f /root/zowe/*.pax && \
    rm -rf /root/zowe/zowe-install && \
    cd /root/zowe/install/components && \
    find */bin -type f -name '*.sh' -exec sh -c "sed -i 's/-Xquickstart//' {}" \;

# Could be modified during runtime, back it up.
RUN cp /root/zowe/instance/instance.env /root/zowe/instance/instance.env.bkp
RUN cp /root/zowe/install/bin/zowe-setup-certificates.env /root/zowe/install/bin/zowe-setup-certificates.env.bkp

ENTRYPOINT ["/root/zowe/run.sh"]