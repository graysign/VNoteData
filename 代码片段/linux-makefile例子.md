# 1调用其他makefile  

```makefile
SUBDIR:=$(dir $(abspath $(lastword $(MAKEFILE_LIST))))
MAKE = make
subsystem:
	cd $(SUBDIR)src && $(MAKE)
	cd $(SUBDIR)test && $(MAKE)
clean:
	cd $(SUBDIR)src && make clean
	cd $(SUBDIR)test && make clean 
```


# 2常规makefile  

```makeflie
DIR:=$(shell pwd)
CC = g++
CFLAGS = -Wall -O2  -Wno-switch
INCLUDE = -I$(DIR)/include/ -I$(DIR)/util/ 
LINKLIB = -lrt -lpthread
TARGET = $(DIR)/../bin/helloworld

%.o:%.c
	$(CC) $(CFLAGS) -c $< -o $@ $(INCLUDE) 

%.o:%.cpp
	$(CC) $(CFLAGS) -c $< -o $@ $(INCLUDE)

SOURCES = $(wildcard \
./common/*.c \
./*.cpp \
./util/*.cpp )

OBJS = $(patsubst %.c,%.o,$(patsubst %.cpp,%.o,$(SOURCES)))

$(TARGET):$(OBJS)
	$(CC) $(CFLAGS) $(OBJS) -o $(TARGET)  -static $(LINKLIB)
	chmod a+x $(TARGET)

.PHONY: clean
clean:
	find $(DIR) -name "*.o" |xargs -i rm -rf {} 
	rm -rf $(DIR)/../bin/helloworld


```