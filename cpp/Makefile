
# Make sure this is always first.
.PHONY: default
default: all

-include local.mk

CXXFLAG = -Wall -Werror -g -O3
LINK.o = $(LINK.cc)

INCLUDE = -I .
LOCAL_LIBS = log/libcert.a log/libdatabase.a log/liblog.a \
             merkletree/libmerkletree.a \
             proto/libproto.a util/libutil.a
LDLIBS = -lpthread -lgflags -lglog -lssl -lcrypto -lldns -lsqlite3 \
         -lprotobuf -levent_core -levent_extra -levent_pthreads

PLATFORM := $(shell uname -s)
ifeq ($(PLATFORM), FreeBSD)
  LDLIBS += -L/usr/local/lib/event2
else
  LDLIBS += -lresolv
endif

ifeq ($(PLATFORM), Darwin)
  LDLIBS += -lboost_system-mt -lboost_thread-mt
else
  LDLIBS += -lboost_system -lboost_thread
endif

# Need OpenSSL >= 1.0.0
ifneq ($(OPENSSLDIR),)
  INCLUDE += -I $(OPENSSLDIR)/include
  LDLIBS += -L $(OPENSSLDIR) -L $(OPENSSLDIR)/lib -Wl,-rpath,$(OPENSSLDIR) \
            -Wl,-rpath,$(OPENSSLDIR)/lib
endif

# Need libevent
ifneq ($(LIBEVENTDIR),)
  INCLUDE += -I $(LIBEVENTDIR)/include
  LDLIBS += -L $(LIBEVENTDIR)/.libs  -Wl,-rpath,$(LIBEVENTDIR)/.libs
endif

# Json library location explicitly specified - link against that.
ifneq ($(JSONCLIBDIR),)
	INCLUDE += -I $(JSONCLIBDIR)/include
	LDLIBS += -L $(JSONCLIBDIR)/lib -Wl,-rpath,$(JSONCLIBDIR)/lib -ljson-c
else
	JSON_C_CFLAGS := $(shell pkg-config --cflags json-c)
	CXXFLAG += $(JSON_C_CFLAGS)
	JSON_C_LIBS := $(shell pkg-config --libs json-c)
	LDLIBS += $(JSON_C_LIBS)
endif

# Need gtest
GTESTDIR ?= /usr/src/gtest
INCLUDE += -I $(GTESTDIR)/include

# Allow user-system overrides
INCLUDE += -I /usr/local/include
LDLIBS += -L /usr/local/lib

CXXFLAGS = $(INCLUDE) $(CXXFLAG) $(LOCAL_CXXFLAGS)

# File lists to simplify the rest of the Makefile

PROTO_TESTS = proto/serializer_test
MERKLETREE_TESTS = merkletree/merkle_tree_test \
                   merkletree/merkle_tree_large_test \
                   merkletree/serial_hasher_test merkletree/tree_hasher_test
LOG_TESTS = log/cert_test log/cert_checker_test \
            log/cert_submission_handler_test log/database_test \
            log/database_large_test log/file_storage_test \
            log/frontend_signer_test log/frontend_test log/log_lookup_test \
            log/signer_verifier_test log/log_signer_test log/tree_signer_test \
            log/logged_certificate_test log/ct_extensions_test
UTIL_TESTS = util/json_wrapper_test
MONITOR_TESTS = monitor/database_test
ALL_TESTS = $(PROTO_TESTS) $(MERKLETREE_TESTS) $(LOG_TESTS) $(UTIL_TESTS) \
	$(MONITOR_TESTS) dns_tests

all: unit_tests client/ct server/ct-server server/ct-dns-server

.DELETE_ON_ERROR:

.PHONY: clean all unit_tests test alltests benchmark clean \
        proto_tests merkletree_tests log_tests client_tests \
	monitor_tests dns_tests

gtest/Makefile: $(GTESTDIR)
	mkdir -p gtest && cd gtest && cmake $(GTESTDIR)

gtest/libgtest.a: gtest/Makefile
	$(MAKE) -C gtest CPPFLAGS="$(CPPFLAGS)" CXXFLAGS="$(CXXFLAGS)" \
	  LDFLAGS="$(LDFLAGS)" gtest

$(ALL_TESTS): gtest/libgtest.a

proto/.depend: proto/*.cc */*.h
	$(CXX) $(CXXFLAGS) -MM -MG proto/*.cc > proto/.depend1
	sed 's,\(.*\)\.o,proto/\1.o,' > proto/.depend < proto/.depend1
	rm proto/.depend1

merkletree/.depend: merkletree/*.cc */*.h
	$(CXX) $(CXXFLAGS) -MM -MG merkletree/*.cc > merkletree/.depend1
	sed 's,\(.*\)\.o,merkletree/\1.o,' > merkletree/.depend \
					   < merkletree/.depend1
	rm merkletree/.depend1

log/.depend: log/*.cc */*.h
	$(CXX) $(CXXFLAGS) -MM -MG log/*.cc > log/.depend1
	sed 's,\(.*\)\.o,log/\1.o,' > log/.depend < log/.depend1
	rm log/.depend1

util/.depend: util/*.cc */*.h
	$(CXX) $(CXXFLAGS) -MM -MG util/*.cc > util/.depend1
	sed 's,\(.*\)\.o,util/\1.o,' > util/.depend < util/.depend1
	rm util/.depend1

monitor/.depend: monitor/*.cc */*.h
	$(CXX) $(CXXFLAGS) -MM -MG monitor/*.cc > monitor/.depend1
	sed 's,\(.*\)\.o,monitor/\1.o,' > monitor/.depend < monitor/.depend1
	rm monitor/.depend1

client/.depend: client/*.cc */*.h
	$(CXX) $(CXXFLAGS) -MM -MG client/*.cc > client/.depend1
	sed 's,\(.*\)\.o,client/\1.o,' > client/.depend < client/.depend1
	rm client/.depend1

server/.depend: server/*.cc */*.h
	$(CXX) $(CXXFLAGS) -MM -MG server/*.cc > server/.depend1
	sed 's,\(.*\)\.o,server/\1.o,' > server/.depend < server/.depend1
	rm server/.depend1

### proto preprocessing
proto/%.pb.h proto/%.pb.cc: ../proto/%.proto
	protoc -I .. $^ --cpp_out=.

ifneq ($(MAKECMDGOALS),clean)
    include proto/.depend
    include merkletree/.depend
    include log/.depend
    include util/.depend
    include monitor/.depend
    include client/.depend
    include server/.depend
endif

unit_tests: proto_tests merkletree_tests log_tests util_tests monitor_tests

util_tests: util/json_wrapper_test

### util/ targets
util/libutil.a: util/util.o util/openssl_util.o util/testing.o \
                util/json_wrapper.o util/thread_pool.o util/libevent_wrapper.o
	rm -f $@
	ar -rcs $@ $^

util/json_wrapper_test: util/json_wrapper_test.o util/libutil.a

### proto/ targets
proto/libproto.a: proto/ct.pb.o proto/serializer.o
	rm -f $@
	ar -rcs $@ $^

proto_tests: $(PROTO_TESTS)

proto/serializer_test: proto/serializer_test.o proto/libproto.a util/libutil.a

### merkletree/ targets
merkletree/libmerkletree.a: merkletree/compact_merkle_tree.o \
                            merkletree/merkle_tree.o \
                            merkletree/merkle_tree_math.o \
                            merkletree/merkle_verifier.o \
                            merkletree/serial_hasher.o merkletree/tree_hasher.o
	rm -f $@
	ar -rcs $@ $^

merkletree_tests: $(MERKLETREE_TESTS)

merkletree/merkle_tree_large_test: merkletree/merkle_tree_large_test.o \
                                   merkletree/libmerkletree.a util/libutil.a

merkletree/merkle_tree_test: merkletree/merkle_tree_test.o \
                             merkletree/libmerkletree.a util/libutil.a

merkletree/serial_hasher_test: merkletree/serial_hasher_test.o \
                               merkletree/serial_hasher.o util/libutil.a

merkletree/tree_hasher_test: merkletree/tree_hasher_test.o \
                             merkletree/serial_hasher.o \
                             merkletree/tree_hasher.o util/libutil.a

### log/ targets
log/libcert.a: log/cert.o log/cert_checker.o log/cert_submission_handler.o \
               log/ct_extensions.o merkletree/serial_hasher.o
	rm -f $@
	ar -rcs $@ $^

log/libdatabase.a: log/file_storage.o log/filesystem_op.o log/file_db_cert.o \
                   log/sqlite_db_cert.o
	rm -f $@
	ar -rcs $@ $^

log/liblog.a: log/log_signer.o log/signer.o log/verifier.o log/frontend.o \
              log/frontend_signer.o log/log_verifier.o log/tree_signer_cert.o \
              log/log_lookup_cert.o
	rm -f $@
	ar -rcs $@ $^

log_tests: $(LOG_TESTS)

log/cert_test: log/cert_test.o log/cert.o log/ct_extensions.o util/libutil.a \
               merkletree/serial_hasher.o log/frontend.o proto/ct.pb.o \
               log/frontend_signer.o log/cert_submission_handler.o \
               log/log_signer.o log/signer.o log/verifier.o proto/serializer.o \
               log/cert_checker.o

log/cert_checker_test: log/cert_checker_test.o log/cert.o log/cert_checker.o \
                       log/ct_extensions.o util/libutil.a \
                       merkletree/serial_hasher.o log/frontend.o proto/ct.pb.o \
                       log/frontend_signer.o log/cert_submission_handler.o \
                       log/log_signer.o log/signer.o log/verifier.o \
                       proto/serializer.o log/cert_checker.o

log/cert_submission_handler_test: log/cert_submission_handler_test.o \
                                  log/libcert.a util/libutil.a \
                                  log/frontend.o log/frontend_signer.o \
                                  proto/libproto.a log/log_signer.o \
                                  log/signer.o log/verifier.o

log/ct_extensions_test: log/ct_extensions_test.o log/libcert.a util/libutil.a \
                        log/liblog.a proto/ct.pb.o \
                        log/cert_submission_handler.o proto/serializer.o \
                        log/cert_checker.o

log/database_large_test: log/database_large_test.o log/libdatabase.a \
                         log/log_signer.o log/signer.o log/verifier.o \
                         log/test_signer.o merkletree/libmerkletree.a \
                         proto/libproto.a util/libutil.a

log/database_test: log/database_test.o log/libdatabase.a \
                   log/log_signer.o log/signer.o log/verifier.o \
                   log/test_signer.o merkletree/libmerkletree.a \
                   proto/libproto.a util/libutil.a

log/file_storage_test: log/file_storage_test.o log/libdatabase.a \
                       proto/libproto.a util/libutil.a \
                       merkletree/libmerkletree.a

log/frontend_signer_test: log/frontend_signer_test.o \
                          log/frontend_signer.o \
                          log/log_signer.o log/signer.o log/verifier.o \
                          log/log_verifier.o log/test_signer.o log/libcert.a \
                          log/libdatabase.a merkletree/libmerkletree.a \
                          proto/libproto.a util/libutil.a

log/frontend_test: log/frontend_test.o log/frontend.o \
                   log/frontend_signer.o \
                   log/log_signer.o log/signer.o log/verifier.o \
                   log/log_verifier.o log/test_signer.o \
                   log/libcert.a log/libdatabase.a merkletree/libmerkletree.a \
                   proto/libproto.a util/libutil.a

log/log_lookup_test: log/log_lookup_test.o log/test_signer.o log/libdatabase.a \
                     log/liblog.a merkletree/libmerkletree.a proto/libproto.a \
                     util/libutil.a

log/log_signer_test: log/log_signer_test.o log/log_signer.o log/signer.o \
                     log/verifier.o log/test_signer.o \
                     merkletree/libmerkletree.a proto/libproto.a util/libutil.a

log/signer_verifier_test: log/signer_verifier_test.o log/log_signer.o \
                     log/signer.o log/verifier.o log/test_signer.o \
                     merkletree/libmerkletree.a proto/libproto.a util/libutil.a

log/tree_signer_test: log/tree_signer_test.o log/log_signer.o log/signer.o \
                      log/verifier.o log/test_signer.o log/tree_signer_cert.o \
                      log/log_verifier.o \
                      log/libdatabase.a merkletree/libmerkletree.a \
                      proto/libproto.a util/libutil.a

log/logged_certificate_test: log/logged_certificate_test.o proto/libproto.a \
                             util/libutil.a merkletree/libmerkletree.a

monitor_tests: $(MONITOR_TESTS)

monitor/database_test: monitor/database_test.o monitor/database.o \
                       monitor/sqlite_db.o util/libutil.a log/test_signer.o \
                       merkletree/libmerkletree.a log/log_signer.o \
                       log/signer.o log/verifier.o proto/libproto.a

# client
client/ct: client/ct.o client/client.o client/ssl_client.o \
           client/http_log_client.o monitor/sqlite_db.o monitor/database.o \
           monitor/monitor.o client/async_log_client.o \
           $(LOCAL_LIBS)

server/ct-dns-server: server/ct-dns-server.o server/event.o $(LOCAL_LIBS)

server/ct-server: server/ct-server.o util/read_private_key.o \
	server/handler.o $(LOCAL_LIBS)

dns_tests: server/ct-dns-server

test: all
	util/json_wrapper_test
	proto/serializer_test
	merkletree/serial_hasher_test
	merkletree/tree_hasher_test
	merkletree/merkle_tree_test
# Do not run merkletree/merkle_tree_large_test by default
	log/logged_certificate_test
	log/cert_test --test_certs_dir=../test/testdata
	log/cert_checker_test --test_certs_dir=../test/testdata
	log/cert_submission_handler_test --test_certs_dir=../test/testdata
	log/ct_extensions_test --test_certs_dir=../test/testdata
	log/file_storage_test
	log/database_test
# Do not run log/database_large_test by default
	log/log_signer_test
	log/frontend_signer_test
	log/frontend_test --test_certs_dir=../test/testdata
	log/tree_signer_test
	log/log_lookup_test
	monitor/database_test
# TODO(pphaneuf): ct-dns-server-test is broken at the moment.
#	python server/ct-dns-server-test.py

# Unit tests plus end-to-end tests. Make sure to set up links in test/  first.
alltests: test
	$(MAKE) -C test test

benchmark: merkletree/merkle_tree_large_test log/database_large_test
	merkletree/merkle_tree_large_test
	@echo "----- Running database large test with --database_size=100 -----"
	log/database_large_test --database_size=100
	@echo "If you want to test other sizes, run log/database_large_test \
	with --database_size=x --batch_size=y"

clean:
	find . -name '*.[o|a]' | xargs rm -f
	find . -name '*_test' | xargs rm -f
	rm -f proto/*.pb.h proto/*.pb.cc */.depend*
	rm -rf gtest/*
