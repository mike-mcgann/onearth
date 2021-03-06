GDAL_VERSION=1.11.0
GDAL_ARTIFACT=gdal-$(GDAL_VERSION).tar.gz
GDAL_HOME=http://download.osgeo.org/gdal
GDAL_URL=$(GDAL_HOME)/$(GDAL_VERSION)/$(GDAL_ARTIFACT)

ONEARTH_VERSION=0.5.0

POSTGRES_VERSION=9.2

PREFIX=/usr/local
SMP_FLAGS=-j $(shell cat /proc/cpuinfo | grep processor | wc -l)
LIB_DIR=$(shell \
	[ "$(shell arch)" == "x86_64" ] \
		&& echo "lib64" \
		|| echo "lib" \
)
RPMBUILD_FLAGS=-ba

all: 
	@echo "Use targets gdal-rpm or onearth-rpm"

gdal: gdal-unpack mrf-overlay gdal-patch gdal-compile

onearth: onearth-compile

#-----------------------------------------------------------------------------
# Download
#-----------------------------------------------------------------------------

download: gdal-download 

gdal-download: upstream/$(GDAL_ARTIFACT).downloaded

upstream/$(GDAL_ARTIFACT).downloaded: 
	mkdir -p upstream
	rm -f upstream/$(GDAL_ARTIFACT)
	( cd upstream ; wget $(GDAL_URL) )
	touch upstream/$(GDAL_ARTIFACT).downloaded

#-----------------------------------------------------------------------------
# Compile
#-----------------------------------------------------------------------------

gdal-unpack: build/gdal/VERSION

build/gdal/VERSION:
	mkdir -p build/gdal
	tar xf upstream/$(GDAL_ARTIFACT) -C build/gdal \
		--strip-components=1 --exclude=.gitignore

mrf-overlay:
	cp -r src/gdal_mrf/* build/gdal

gdal-patch: build/gdal/swig/python/GNUmakefile

build/gdal/swig/python/GNUmakefile: deploy/gibs-gdal/python-install.patch
# 	Python install does not respect DESTDIR
	( cd build/gdal ; \
		patch -p0 < ../../deploy/gibs-gdal/python-install.patch )
# 	Use external libtool
	sed -i 's|@LIBTOOL@|/usr/bin/libtool|g' build/gdal/GDALmake.opt.in

#   Build MRF into GDAL
	sed -i 's|GDAL_FORMATS = |GDAL_FORMATS = mrf |g' build/gdal/GDALmake.opt.in
	sed -i -e'/^CPL_C_START/a void CPL_DLL GDALRegister_mrf(void);' build/gdal/gcore/gdal_frmts.h
	sed -i -e'/AutoLoadDrivers/a #ifdef FRMT_mrf\n    GDALRegister_mrf();\n#endif' build/gdal/frmts/gdalallregister.cpp

#	Patch gcore/overview.cpp	
	( cd build/gdal/gcore ; \
		patch < ../../../deploy/gibs-gdal/overview.patch )

gdal-compile:
	( cd build/gdal ; ./configure \
		--prefix=$(PREFIX) \
		--libdir=$(PREFIX)/$(LIB_DIR) \
		--mandir=$(PREFIX)/share/man \
		--with-threads \
		--without-bsb \
		--with-geotiff=internal \
		--with-libtiff=internal \
		--without-ogdi \
		--with-libz \
		--with-netcdf \
		--with-hdf4 \
		--with-hdf5 \
		--with-geos \
		--with-jasper \
		--with-png \
		--with-gif \
		--with-jpeg \
		--with-odbc \
		--with-sqlite3 \
		--with-mysql \
		--with-curl \
		--with-python=yes \
		--with-pcraster \
		--with-xerces \
		--with-xerces-lib='-lxerces-c' \
		--with-xerces-inc=/usr/include \
		--with-jpeg12=no \
		--enable-shared \
		--with-gdal-ver=$(GDAL_VERSION) \
		--disable-rpath \
		--with-pg=/usr/pgsql-$(POSTGRES_VERSION)/bin/pg_config \
		--with-expat \
	)
	$(MAKE) -C build/gdal $(SMP_FLAGS) all man
	$(MAKE) -C build/gdal/frmts/mrf plugin

onearth-compile:
	$(MAKE) -C src/mod_onearth \
		LIBS=-L/usr/pgsql-$(POSTGRES_VERSION)/lib \
		LDFLAGS=-lpq

#-----------------------------------------------------------------------------
# Install
#-----------------------------------------------------------------------------
install: gdal-install onearth-install 

gdal-install:
	$(MAKE) -C build/gdal install install-man PREFIX=$(PREFIX)
	$(MAKE) -C build/gdal/my_apps install

onearth-install:
	install -m 755 -d $(DESTDIR)/$(PREFIX)/$(LIB_DIR)/httpd/modules
	install -m 755 src/mod_onearth/.libs/mod_twms.so \
		$(DESTDIR)/$(PREFIX)/$(LIB_DIR)/httpd/modules/mod_twms.so
	install -m 755 src/mod_onearth/.libs/mod_wms.so \
		$(DESTDIR)/$(PREFIX)/$(LIB_DIR)/httpd/modules/mod_wms.so

	install -m 755 -d $(DESTDIR)/$(PREFIX)/bin
	install -m 755 src/mod_onearth/twms_tool \
		$(DESTDIR)/$(PREFIX)/bin/oe_create_cache_config
	install -m 755 src/layer_config/bin/oe_configure_layer.py  \
		-D $(DESTDIR)/$(PREFIX)/bin/oe_configure_layer
	install -m 755 src/onearth_logs/onearth_logs.py  \
		-D $(DESTDIR)/$(PREFIX)/bin/onearth_metrics
	install -m 755 src/generate_legend/oe_generate_legend.py  \
		-D $(DESTDIR)/$(PREFIX)/bin/oe_generate_legend.py
	install -m 755 src/mrfgen/mrfgen.py  \
		-D $(DESTDIR)/$(PREFIX)/bin/mrfgen
	install -m 755 src/mrfgen/colormap2vrt.py  \
		-D $(DESTDIR)/$(PREFIX)/bin/colormap2vrt.py
	install -m 755 src/mrfgen/RGBApng2Palpng  \
		-D $(DESTDIR)/$(PREFIX)/bin/RGBApng2Palpng

	install -m 755 -d $(DESTDIR)/$(PREFIX)/share/onearth
	install -m 755 -d $(DESTDIR)/$(PREFIX)/share/onearth/apache
	install -m 755 -d $(DESTDIR)/$(PREFIX)/share/onearth/apache/kml
	install -m 755 src/cgi/twms.cgi \
		-t $(DESTDIR)/$(PREFIX)/share/onearth/apache
	install -m 755 src/cgi/wmts.cgi \
		-t $(DESTDIR)/$(PREFIX)/share/onearth/apache
	cp src/cgi/kml/* \
		-t $(DESTDIR)/$(PREFIX)/share/onearth/apache/kml
	cp src/cgi/index.html \
		-t $(DESTDIR)/$(PREFIX)/share/onearth/apache
	cp src/mrfgen/empty_tiles/black.jpg \
		-t $(DESTDIR)/$(PREFIX)/share/onearth/apache
	cp src/mrfgen/empty_tiles/transparent.png \
		-t $(DESTDIR)/$(PREFIX)/share/onearth/apache

	install -m 755 -d $(DESTDIR)/$(PREFIX)/share/onearth/mrfgen
	cp src/mrfgen/empty_tiles/* \
		$(DESTDIR)/$(PREFIX)/share/onearth/mrfgen

	install -m 755 -d $(DESTDIR)/etc/onearth/config
	cp -r src/layer_config/conf \
		$(DESTDIR)/etc/onearth/config
	cp -r src/layer_config/layers \
		$(DESTDIR)/etc/onearth/config
	cp -r src/layer_config/schema \
		$(DESTDIR)/etc/onearth/config
	install -m 755 -d $(DESTDIR)/etc/onearth/config/headers

	install -m 755 -d $(DESTDIR)/etc/onearth/metrics
	cp -r src/onearth_logs/logs.* \
		$(DESTDIR)/etc/onearth/metrics
	cp -r src/onearth_logs/tilematrixsetmap.* \
		$(DESTDIR)/etc/onearth/metrics

	install -m 755 -d $(DESTDIR)/$(PREFIX)/share/onearth/demo
	cp -r src/demo/* $(DESTDIR)/$(PREFIX)/share/onearth/demo


#-----------------------------------------------------------------------------
# Local install
#-----------------------------------------------------------------------------
local-install: gdal-local-install onearth-local-install

gdal-local-install: 
	mkdir -p build/install
	$(MAKE) gdal-install DESTDIR=$(PWD)/build/install

onearth-local-install: 
	mkdir -p build/install
	$(MAKE) onearth-install DESTDIR=$(PWD)/build/install

#-----------------------------------------------------------------------------
# Artifacts
#-----------------------------------------------------------------------------
artifacts: gdal-artifact onearth-artifact

gdal-artifact: 
	mkdir -p dist
	rm -f dist/gibs-gdal-$(GDAL_VERSION).tar.bz2
	tar cjvf dist/gibs-gdal-$(GDAL_VERSION).tar.bz2 \
		--transform="s,^,gibs-gdal-$(GDAL_VERSION)/," \
		src/gdal_mrf deploy/gibs-gdal GNUmakefile

onearth-artifact: onearth-clean
	mkdir -p dist
	rm -rf dist/onearth-$(ONEARTH_VERSION).tar.bz2
	tar cjvf dist/onearth-$(ONEARTH_VERSION).tar.bz2 \
		--transform="s,^,onearth-$(ONEARTH_VERSION)/," \
		src/mod_onearth src/layer_config src/mrfgen src/cgi \
		src/demo src/onearth_logs src/generate_legend GNUmakefile

#-----------------------------------------------------------------------------
# RPM
#-----------------------------------------------------------------------------
rpm: gdal-rpm onearth-rpm

gdal-rpm: gdal-artifact 
	mkdir -p build/rpmbuild/SOURCES
	mkdir -p build/rpmbuild/BUILD	
	mkdir -p build/rpmbuild/BUILDROOT
	rm -f dist/gibs-gdal*.rpm
	cp \
		upstream/gdal-$(GDAL_VERSION).tar.gz \
		dist/gibs-gdal-$(GDAL_VERSION).tar.bz2 \
		build/rpmbuild/SOURCES
	rpmbuild \
		--define _topdir\ "$(PWD)/build/rpmbuild" \
		-ba deploy/gibs-gdal/gibs-gdal.spec 
	mv build/rpmbuild/RPMS/*/gibs-gdal*.rpm dist
	mv build/rpmbuild/SRPMS/gibs-gdal*.rpm dist

onearth-rpm: onearth-artifact 
	mkdir -p build/rpmbuild/SOURCES
	mkdir -p build/rpmbuild/BUILD	
	mkdir -p build/rpmbuild/BUILDROOT
	rm -f dist/onearth*.rpm
	cp \
		dist/onearth-$(ONEARTH_VERSION).tar.bz2 \
		build/rpmbuild/SOURCES
	rpmbuild \
		--define _topdir\ "$(PWD)/build/rpmbuild" \
		-ba deploy/onearth/onearth.spec 
	mv build/rpmbuild/RPMS/*/onearth*.rpm dist

#-----------------------------------------------------------------------------
# Mock
#-----------------------------------------------------------------------------
mock: gdal-mock onearth-mock

gdal-mock:
	mock --clean
	mock --root=gibs-epel-6-$(shell arch) \
		dist/gibs-gdal-$(GDAL_VERSION)-*.src.rpm

onearth-mock:
	mock --clean
	mock --init
	mock --copyin dist/gibs-gdal-*$(GDAL_VERSION)-*.$(shell arch).rpm /
	mock --install yum
	mock --shell \
	       "yum install -y /gibs-gdal-*$(GDAL_VERSION)-*.$(shell arch).rpm"
	mock --rebuild --no-clean \
		dist/mod_twms-$(ONEARTH_VERSION)-*.src.rpm

#-----------------------------------------------------------------------------
# Clean
#-----------------------------------------------------------------------------
clean: onearth-clean
	rm -rf build

onearth-clean:
	$(MAKE) -C src/mod_onearth clean

distclean: clean
	rm -rf dist
	rm -rf upstream


