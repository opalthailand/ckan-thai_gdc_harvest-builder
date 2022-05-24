FROM alpine:3.7

# Internal environment variables
ENV APP_DIR=/srv/app
ENV SRC_DIR=/srv/app/src
ENV CKAN_INI=${APP_DIR}/ckan.ini
ENV PIP_SRC=${SRC_DIR}
ENV CKAN_STORAGE_PATH=/var/lib/ckan
ENV GIT_URL=https://github.com/ckan/ckan.git
# CKAN version to build
ENV GIT_BRANCH=ckan-2.9.5
# Customize these on the .env file if needed
ENV CKAN_SITE_URL=http://localhost:5000
ENV CKAN__PLUGINS image_view text_view recline_view datastore datapusher envvars

WORKDIR ${APP_DIR}

# Install necessary packages to run CKAN
RUN apk add --no-cache tzdata \
        git \
        gettext \
        postgresql-client \
        python \
        apache2-utils \
        libxml2 \
        libxslt \
        musl-dev \
        uwsgi-http \
        uwsgi-corerouter \
        uwsgi-python \
        py2-gevent \
        uwsgi-gevent \
        libmagic \
        curl \
        sudo && \
    # Packages to build CKAN requirements and plugins
    apk add --no-cache --virtual .build-deps \
        postgresql-dev \
        gcc \
        make \
        g++ \
        autoconf \
        automake \
	    libtool \
        python-dev \
        py2-virtualenv \
        libxml2-dev \
        libxslt-dev \
        linux-headers && \
    # Create SRC_DIR
    mkdir -p ${SRC_DIR} && \
    # curl -o ${SRC_DIR}/get-pip.py https://bootstrap.pypa.io/2.7/get-pip.py && \
    curl -o ${SRC_DIR}/get-pip.py https://bootstrap.pypa.io/pip/2.7/get-pip.py && \
    python ${SRC_DIR}/get-pip.py 'pip==20.3.3' && \     
    rm -rf ${SRC_DIR}/get-pip.py

# Set up Python3 virtual environment
# RUN cd ${APP_DIR} && \
#     python -m venv ${APP_DIR} && \
#     source ${APP_DIR}/bin/activate
# Set up python2 virtual environment
RUN virtualenv --no-site-packages ${APP_DIR} && \
    source ${APP_DIR}/bin/activate

# # Virtual environment binaries/scripts to be used first
ENV PATH=${APP_DIR}/bin:${PATH}  

RUN pip install -e git+${GIT_URL}@${GIT_BRANCH}#egg=ckan[requirements-py2] && \ 
    # fix ssl error
    git config --global http.sslVerify "false" && \
    pip install uwsgi && \
    cd ${SRC_DIR}/ckan && \
    cp who.ini ${APP_DIR} && \
    pip install -e git+https://github.com/okfn/ckanext-envvars.git#egg=ckanext-envvars && \
    pip install -e git+https://github.com/ckan/ckanext-scheming.git@367d50f#egg=ckanext-scheming && \
    pip install -r ${SRC_DIR}/ckanext-scheming/requirements.txt && \
    # pip install -e git+https://gitlab.nectec.or.th/opend/ckanext-pdfview.git#egg=ckanext-pdfview && \
    pip install -e git+https://github.com/ckan/ckanext-pdfview.git#egg=ckanext-pdfview && \
    pip install -e git+https://github.com/davidread/ckanext-hierarchy.git#egg=ckanext-hierarchy && \
    pip install -r ${SRC_DIR}/ckanext-hierarchy/requirements.txt && \
    pip install -e git+https://github.com/ckan/ckanext-dcat.git#egg=ckanext-dcat && \
    pip install -r ${SRC_DIR}/ckanext-dcat/requirements.txt && \
    pip install -e git+https://gitlab.nectec.or.th/opend/ckanext-xloader.git#egg=ckanext-xloader && \
    # pip install -e git+https://gitlab.nectec.or.th/opend/ckanext-xloader.git#egg=ckanext-xloader && \
    pip install -r ${SRC_DIR}/ckanext-xloader/requirements.txt && \
    pip install -r ${SRC_DIR}/ckanext-xloader/dev-requirements.txt && \
    pip install -e git+https://gitlab.nectec.or.th/opend/ckanext-dga-stats.git#egg=ckanext-dga-stats && \
    pip install -e git+https://gitlab.nectec.or.th/opend/ckanext-thai_gdc.git#egg=ckanext-thai_gdc && \
    # ADDDDDDDDDDDD
    pip install pandas && \
    # Create and update CKAN config
    ckan generate config ${CKAN_INI}

# Install and configure supervisor
RUN pip install supervisor && \
mkdir /etc/supervisord.d

# Copy all setup files
COPY ./setup ${APP_DIR}
COPY ./setup/supervisor.worker.conf /etc/supervisord.d/worker.conf
COPY ./setup/supervisor.cron.conf /etc/supervisord.d/cron.conf
COPY ./setup/supervisord.conf /etc/supervisord.conf

# Create a local user and group to run the app
RUN addgroup -g 92 -S ckan && \
    adduser -u 92 -h /srv/app -H -D -S -G ckan ckan

# Create local storage folder
RUN mkdir -p $CKAN_STORAGE_PATH && \
    chown -R ckan:ckan $CKAN_STORAGE_PATH

# Create ckan cron job file
RUN echo "0 * * * * ckan -c /srv/app/ckan.ini tracking update >> /var/log/ckan_tracking.log && ckan -c /srv/app/ckan.ini search-index rebuild -r >> /var/log/ckan_rebuild.log" > /etc/crontabs/root

##### ADD
RUN apk add libffi-dev
RUN . /srv/app/bin/activate && \
pip install cryptography==2.8  && \
pip install -e git+https://github.com/ckan/ckanext-harvest.git#egg=ckanext-harvest && \
cd /srv/app/src/ckanext-harvest && pip install -r pip-requirements.txt

RUN apk add supervisor 
RUN mkdir /var/log/ckan
COPY ckan_harvesting.conf /etc/supervisord.d/
RUN apk add sed
RUN sed -i "22i ckan.harvest.mq.type = redis" ckan.ini
RUN crontab -l | { cat; echo "*/15 * * * * /srv/app/bin/bin/ckan -c /srv/app/bin/ckan.ini harvester run"; } | crontab -

# Create entrypoint directory for children image scripts
ONBUILD RUN mkdir /docker-entrypoint.d
RUN chown ckan -R /srv/app
EXPOSE 5000
HEALTHCHECK --interval=10s --timeout=5s --retries=5 CMD curl --fail http://localhost:5000/api/3/action/status_show || exit 1
ENTRYPOINT ["/srv/app/start_ckan.sh"]

# FROM thepaeth/ckan-thai_gdc:ckan-2.9-xloader

# RUN apk add libffi-dev
# RUN . /srv/app/bin/activate && \
# pip install cryptography==2.8  && \
# pip install -e git+https://github.com/ckan/ckanext-harvest.git#egg=ckanext-harvest && \
# cd /srv/app/src/ckanext-harvest && pip install -r pip-requirements.txt

# RUN apk add supervisor 
# RUN mkdir /var/log/ckan
# COPY ckan_harvesting.conf /etc/supervisord.d/
# RUN apk add sed 

# RUN sed -i "22i ckan.harvest.mq.type = redis" ckan.ini
# RUN sed -i "124i ckan.plugins = envvars stats image_view text_view recline_view resource_proxy webpage_view datastore xloader thai_gdc scheming_datasets pdf_view hierarchy_display hierarchy_form dcat dcat_json_interface structured_data dga_stats harvest ckan_harvester" ckan.ini
# . /srv/app/bin/activate && \
# ckan -c /srv/app/ckan.ini harvester initdb
# RUN crontab -l | { cat; echo "*/15 * * * * /srv/app/bin/bin/ckan -c /srv/app/bin/ckan.ini harvester run"; } | crontab -
# sudo supervisorctl reload
# sudo supervisorctl status