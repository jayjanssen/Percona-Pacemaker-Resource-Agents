SOURCES := $(wildcard *.c *.h)
TARGETS := libybinlogp.so.1 libybinlogp.so ybinlogp prm_binlog_parser

prefix := /usr

CFLAGS += -Wall -ggdb -Wextra --std=c99 -pedantic
LDFLAGS += -L. -lssl
#CFLAGS=-Wall -ggdb -Wextra -DDEBUG

all: $(TARGETS)

ybinlogp: ybinlogp.o libybinlogp.so
	gcc $(CFLAGS) $(LDFLAGS) -o $@ -lybinlogp $<

prm_binlog_parser: prm_binlog_parser.o libybinlogp.so
	gcc $(CFLAGS) $(LDFLAGS) -o $@ -lybinlogp $<
#prm_binlog_parser: prm_binlog_parser.o libybinlogp.o
#	gcc $(CFLAGS) $(LDFLAGS) -o $@ -lybinlogp $<

libybinlogp.so: libybinlogp.so.1
	ln -fs $< $@

libybinlogp.so.1: libybinlogp.o
	gcc $(CFLAGS) $(LDFLAGS) -shared -Wl,-soname,$@ -o $@ $^

libybinlogp.o: libybinlogp.c ybinlogp-private.h
	gcc $(CFLAGS) $(LDFLAGS) -c -fPIC -o $@ $<

clean::
	rm -f $(TARGETS) *.o

ybinlogp.o: ybinlogp.c ybinlogp.h

prm_binlog_parser.o: prm_binlog_parser.c ybinlogp.h
