FROM alpine:3.4

# persistent / runtime deps
ENV PHPIZE_DEPS \
		autoconf \
		file \
		g++ \
		gcc \
		libc-dev \
		make \
		pkgconf \
		re2c
RUN apk add --no-cache --virtual .persistent-deps \
		ca-certificates \
		curl \
		tar \
		apache2 \
		gettext \
		bash \
		xz

ENV PHP_INI_DIR /usr/local/etc/php
RUN mkdir -p $PHP_INI_DIR/conf.d

# Apply stack smash protection to functions using local buffers and alloca()
# Make PHP's main executable position-independent (improves ASLR security mechanism, and has no performance impact on x86_64)
# Enable optimization (-O2)
# Enable linker optimization (this sorts the hash buckets to improve cache locality, and is non-default)
# Adds GNU HASH segments to generated executables (this is used if present, and is much faster than sysv hash; in this configuration, sysv hash is also generated)
# https://github.com/docker-library/php/issues/272
ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2"
ENV PHP_CPPFLAGS="$PHP_CFLAGS"
ENV PHP_LDFLAGS="-Wl,-O1 -Wl,--hash-style=both -pie"
ENV PHP_EXTRA_CONFIGURE_ARGS --with-apxs2

ENV PHP_VERSION 5.3.29
ENV PHP_URL="https://secure.php.net/get/php-${PHP_VERSION}.tar.xz/from/this/mirror" PHP_ASC_URL="https://secure.php.net/get/php-${PHP_VERSION}.tar.xz.asc/from/this/mirror"

RUN set -xe; \
	\
	apk add --no-cache --virtual .fetch-deps \
		gnupg \
		openssl \
	; \
	\
	mkdir -p /usr/src; \
	cd /usr/src; \
	\
	wget -O php.tar.xz "$PHP_URL"; \
	\
	apk del .fetch-deps

COPY ./devops/php/conf/docker-php-source /usr/local/bin/

RUN set -xe \
	&& apk add --no-cache --virtual .build-deps \
		$PHPIZE_DEPS \
		curl-dev \
		libedit-dev \
		libxml2-dev \
		openssl-dev \
		sqlite-dev \
		apache2-dev \
	\
	&& export CFLAGS="$PHP_CFLAGS" \
		CPPFLAGS="$PHP_CPPFLAGS" \
		LDFLAGS="$PHP_LDFLAGS" \
	&& docker-php-source extract \
	&& cd /usr/src/php \
	&& ./configure \
		--with-config-file-path="$PHP_INI_DIR" \
		--with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
		\
		--disable-cgi \
		\
# --enable-ftp is included here because ftp_ssl_connect() needs ftp to be compiled statically (see https://github.com/docker-library/php/issues/236)
		--enable-ftp \
# --enable-mbstring is included here because otherwise there's no way to get pecl to use it properly (see https://github.com/docker-library/php/issues/195)
		--enable-mbstring \
# --enable-mysqlnd is included here because it's harder to compile after the fact than extensions are (since it's a plugin for several extensions, not an extension in itself)
		--enable-mysqlnd \
		\
		--with-curl \
		--with-libedit \
		--with-openssl \
		--with-zlib \
		\
		$PHP_EXTRA_CONFIGURE_ARGS \
	&& make -j "$(getconf _NPROCESSORS_ONLN)" \
	&& make install \
	&& { find /usr/local/bin /usr/local/sbin -type f -perm +0111 -exec strip --strip-all '{}' + || true; } \
	&& make clean \
	&& docker-php-source delete \
	\
	&& runDeps="$( \
		scanelf --needed --nobanner --recursive /usr/local \
			| awk '{ gsub(/,/, "\nso:", $2); print "so:" $2 }' \
			| sort -u \
			| xargs -r apk info --installed \
			| sort -u \
	)" \
	&& apk add --no-cache --virtual .php-rundeps $runDeps \
	\
	&& apk del .build-deps

ENV APACHE_CONFDIR /etc/apache2
ENV APACHE_ENVVARS $APACHE_CONFDIR/envvars

COPY ./devops/apache/conf/envvars $APACHE_ENVVARS

RUN set -ex \
	\
# generically convert lines like
#   export APACHE_RUN_USER=apache
# into
#   : ${APACHE_RUN_USER:=apache}
#   export APACHE_RUN_USER
# so that they can be overridden at runtime ("-e APACHE_RUN_USER=...")
	&& sed -ri 's/^export ([^=]+)=(.*)$/: ${\1:=\2}\nexport \1/' "$APACHE_ENVVARS" \
	\
# setup directories and permissions
	&& . "$APACHE_ENVVARS" \
	&& for dir in \
		"$APACHE_LOCK_DIR" \
		"$APACHE_RUN_DIR" \
		"$APACHE_LOG_DIR" \
		/var/www/html \
	; do \
		rm -rvf "$dir" \
		&& mkdir -p "$dir" \
		&& chown -R "$APACHE_RUN_USER:$APACHE_RUN_GROUP" "$dir"; \
	done \
	&& rm -fr /var/www/localhost

# logs should go to stdout / stderr
RUN set -ex \
	&& . "$APACHE_ENVVARS" \
	&& ln -sfT /dev/stderr "$APACHE_LOG_DIR/error.log" \
	&& ln -sfT /dev/stdout "$APACHE_LOG_DIR/access.log" \
	&& ln -sfT /dev/stdout "$APACHE_LOG_DIR/other_vhosts_access.log"

# PHP files should be handled by PHP, and should be preferred over any other file type
RUN { \
		echo -e '<FilesMatch \.php$>'; \
		echo -e '\tSetHandler application/x-httpd-php'; \
		echo -e '</FilesMatch>'; \
		echo; \
		echo -e 'DirectoryIndex disabled'; \
		echo -e 'DirectoryIndex index.php index.html'; \
		echo;  \
		echo -e '<Directory /var/www/>'; \
		echo -e '\tOptions -Indexes'; \
		echo -e '\tAllowOverride All'; \
		echo -e '</Directory>'; \
	} | tee "$APACHE_CONFDIR/conf.d/php.conf"

# Fix DocumentRoot and PHP module loading
RUN sed -i 's@^DocumentRoot "/var/www/localhost/htdocs"@DocumentRoot "/var/www/html"@' /etc/apache2/httpd.conf \
	&& sed -i 's@<Directory "/var/www/localhost/htdocs">@<Directory "/var/www/html">@' /etc/apache2/httpd.conf \
	&& sed -i 's@LoadModule php5_module        lib/apache2/libphp5.so@LoadModule php5_module        modules/libphp5.so@'  /etc/apache2/httpd.conf

COPY ./devops/php/conf/docker-php-ext-* ./devops/php/docker-php-entrypoint /usr/local/bin/

ENTRYPOINT ["docker-php-entrypoint"]
COPY ./devops/apache/apache2-foreground /usr/local/bin/
COPY ./devops/apache/conf/httpd.conf /usr/local/etc/httpd.conf.tpl

WORKDIR /var/www/html

EXPOSE 80
CMD ["apache2-foreground"]
