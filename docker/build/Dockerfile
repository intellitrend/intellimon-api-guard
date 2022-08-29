FROM scratch

COPY intmon_api_guard /intmon_api_guard

COPY passwd /etc/passwd
USER nobody

ENTRYPOINT [ "/intmon_api_guard" ]

CMD [ "start", "--config", "/intmon_api_guard.yml" ]
