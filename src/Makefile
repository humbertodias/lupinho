# Emscripten Compiler
CC = emcc

# Native Compiler (Linux / macOS)
CC_NATIVE = gcc

# MinGW (native gcc on Windows, cross-compiler elsewhere)
ifeq ($(OS),Windows_NT)
CC_MINGW ?= gcc
AR_MINGW ?= ar
else
CC_MINGW ?= x86_64-w64-mingw32-gcc
AR_MINGW ?= x86_64-w64-mingw32-ar
endif

# Source Files
SRC = main.c ui.c lua_api.c

# Output File
OUTPUT = ../dist/game.html

# Raylib Directories
RAYLIB_SRC = libs/web/raylib-web
RAYLIB_INCLUDE = -Ilibs/web/raylib-web
RAYLIB_LIB = libs/web/raylib-web/libraylib.web.a
RAYLIB_WIN_DIR = libs/windows
RAYLIB_WIN_LIB = $(RAYLIB_WIN_DIR)/libraylib.a
RAYLIB_LINUX_DIR = libs/linux/raylib
RAYLIB_LINUX_LIB = $(RAYLIB_LINUX_DIR)/libraylib.a

# Lua Configuration
LUA_WEB_INCLUDE = -Ilibs/web/lua-web
LUA_WEB_LIB = libs/web/lua-web/liblua.web.a

# Lua 5.5.1 (compiled from source on each native target)
LUA_55_INCLUDE = -Ilibs/linux/lua-5.5.1/src
LUA_55_DIR = libs/linux/lua-5.5.1/src
# lua.c / luac.c are the standalone interpreter and compiler, not the library.
LUA_55_SRC = $(filter-out $(LUA_55_DIR)/lua.c $(LUA_55_DIR)/luac.c,$(wildcard $(LUA_55_DIR)/*.c))

# Native (Linux) Libraries — raylib is built from vendored sources
NATIVE_INCLUDE = $(LUA_55_INCLUDE) $(RAYLIB_INCLUDE)
NATIVE_LIBS = -Wl,--whole-archive $(RAYLIB_LINUX_LIB) -Wl,--no-whole-archive \
	-lGL -lm -lpthread -ldl -lrt -lX11 -lXcursor -lXinerama -lXrandr -lXi -lz

# Native (Linux / macOS / Windows) Source Files
SRC_NATIVE = main.c ui.c lua_api.c zip.c

# macOS Homebrew prefixes
MACOS_INCLUDE = $(LUA_55_INCLUDE) \
	-I/opt/homebrew/include \
	-I/usr/local/include
MACOS_LIBS = \
	-L/opt/homebrew/lib \
	-L/usr/local/lib \
	-lraylib -lz -lm \
	-framework OpenGL -framework Cocoa -framework IOKit -framework CoreVideo

# Windows MinGW
WIN_INCLUDE = $(LUA_55_INCLUDE) $(RAYLIB_INCLUDE)
WIN_LIBS = -Wl,--whole-archive $(RAYLIB_WIN_LIB) -Wl,--no-whole-archive \
	-lopengl32 -lgdi32 -lwinmm -lz -pthread -static-libgcc
WIN_CFLAGS = $(CFLAGS) -D_WIN32_WINNT=0x0601

# Compilation Flags
CFLAGS = -Wall -std=c99 -D_DEFAULT_SOURCE -Wno-missing-braces

# Emscripten Web Flags
EMFLAGS = -s USE_GLFW=3
# Note: ASYNCIFY removed - not needed and causes ~50% performance overhead
EMFLAGS += -s TOTAL_MEMORY=67108864
EMFLAGS += -s STACK_SIZE=1048576
EMFLAGS += -s FORCE_FILESYSTEM=1
EMFLAGS += --shell-file libs/web/raylib-web/minshell.html
EMFLAGS += -DPLATFORM_WEB

EMFLAGS += -s EXPORTED_FUNCTIONS='["_main", "_lupi_load_game", "_malloc", "_free"]'
EMFLAGS += -s EXPORTED_RUNTIME_METHODS='["ccall", "FS"]'

# Optimization
OPTIMIZATION = -O2

# Debug/Production Flags
# Production flags without closure compiler (causes issues with Raylib)
DEBUG_FLAGS = -DDEBUG_MODE
PROD_FLAGS = -O3 -s ASSERTIONS=0 -s DISABLE_EXCEPTION_CATCHING=1 -s ELIMINATE_DUPLICATE_FUNCTIONS=1 -DPRODUCTION

# Web Target (development with debug)
web: $(SRC)
	$(CC) $(SRC) -o $(OUTPUT) $(RAYLIB_INCLUDE) $(LUA_WEB_INCLUDE) $(CFLAGS) $(EMFLAGS) $(OPTIMIZATION) $(LUA_WEB_LIB) $(RAYLIB_LIB)

# Production Target (optimized, no debug)
production: $(SRC)
	$(CC) $(SRC) -o $(OUTPUT) $(RAYLIB_INCLUDE) $(LUA_WEB_INCLUDE) $(CFLAGS) $(EMFLAGS) $(PROD_FLAGS) $(LUA_WEB_LIB) $(RAYLIB_LIB)

# Web Game Target (embed .lupi file)
web-game: $(SRC)
	@if [ -z "$(GAME_PATH)" ]; then echo "Error: GAME_PATH is required. Usage: make web-game GAME_PATH=game.lupi"; exit 1; fi
	@echo "Extracting $(GAME_PATH)..."
	@rm -rf /tmp/lupi-web-game
	@mkdir -p /tmp/lupi-web-game
	@unzip -q "$(GAME_PATH)" -d /tmp/lupi-web-game -x "*.git*" "*.lupi"
	@echo "Building with embedded game..."
	@mkdir -p ../dist/webgame
	$(CC) $(SRC) -o ../dist/webgame/game.html $(RAYLIB_INCLUDE) $(LUA_WEB_INCLUDE) $(CFLAGS) $(EMFLAGS) $(OPTIMIZATION) -DAUTO_LOAD_GAME $(LUA_WEB_LIB) $(RAYLIB_LIB) --preload-file /tmp/lupi-web-game@/loaded_game
	@echo "Build complete! Output in ../dist/webgame/"

# Raylib static library for Linux
$(RAYLIB_LINUX_LIB):
	@mkdir -p $(RAYLIB_LINUX_DIR)
	$(MAKE) -C $(RAYLIB_SRC) raylib \
		PLATFORM=PLATFORM_DESKTOP \
		RAYLIB_LIBTYPE=STATIC \
		RAYLIB_SRC_PATH=. \
		RAYLIB_RELEASE_PATH="$(CURDIR)/$(RAYLIB_LINUX_DIR)"

# Linux Target (native desktop)
linux: $(SRC_NATIVE) $(LUA_55_SRC) $(RAYLIB_LINUX_LIB)
	@mkdir -p ../dist
	$(CC_NATIVE) $(SRC_NATIVE) $(LUA_55_SRC) -o ../dist/lupinho \
		$(NATIVE_INCLUDE) $(CFLAGS) -DLUA_USE_POSIX -DLUA_USE_DLOPEN \
		$(NATIVE_LIBS) $(OPTIMIZATION)

# macOS Target (native desktop; compiles Lua for Darwin)
macos: $(SRC_NATIVE) $(LUA_55_SRC)
	@mkdir -p ../dist
	$(CC_NATIVE) $(SRC_NATIVE) $(LUA_55_SRC) -o ../dist/lupinho \
		$(MACOS_INCLUDE) $(CFLAGS) -DLUA_USE_POSIX \
		$(MACOS_LIBS) $(OPTIMIZATION)

# Linux Static Target (portable binary with static raylib)
static: linux
	@cp ../dist/lupinho ../dist/lupinho-static
	strip ../dist/lupinho-static

# Raylib static library for Windows (MinGW)
$(RAYLIB_WIN_LIB):
	@mkdir -p $(RAYLIB_WIN_DIR)
	$(MAKE) -C $(RAYLIB_SRC) raylib \
		OS=Windows_NT \
		PLATFORM_SHELL=sh \
		PLATFORM=PLATFORM_DESKTOP \
		RAYLIB_LIBTYPE=STATIC \
		CC="$(CC_MINGW)" \
		AR="$(AR_MINGW)" \
		RAYLIB_SRC_PATH=. \
		RAYLIB_RELEASE_PATH="$(CURDIR)/$(RAYLIB_WIN_DIR)"

# Windows Target (MinGW desktop, cross-compile from Linux/macOS or native MinGW)
windows: $(SRC_NATIVE) $(LUA_55_SRC) $(RAYLIB_WIN_LIB)
	@mkdir -p ../dist
	$(CC_MINGW) $(SRC_NATIVE) $(LUA_55_SRC) -o ../dist/lupinho.exe \
		$(WIN_INCLUDE) $(WIN_CFLAGS) $(OPTIMIZATION) \
		$(WIN_LIBS)

# Clean generated files
clean:
	rm -rf ../dist/webgame/
	rm -f $(OUTPUT) game.js game.wasm lupi_emulator ../dist/lupinho ../dist/lupinho-static ../dist/lupinho.exe
	rm -f $(RAYLIB_WIN_LIB) $(RAYLIB_LINUX_LIB) $(RAYLIB_SRC)/*.o

# Help
help:
	@echo "Lupi Emulator Makefile"
	@echo "======================"
	@echo ""
	@echo "Targets:"
	@echo "  make web         - Build for WebAssembly (development mode with debug)"
	@echo "  make production  - Build for WebAssembly (optimized, no debug)"
	@echo "  make web-game GAME_PATH=game.lupi - Build for Web with embedded .lupi game (output: dist/webgame/)"
	@echo "  make linux       - Build for Linux desktop (vendored raylib + Lua)"
	@echo "  make macos       - Build for macOS desktop (requires raylib via Homebrew)"
	@echo "  make windows     - Build lupinho.exe with MinGW (cross or native)"
	@echo "  make static      - Alias of linux with stripped binary"
	@echo "  make clean       - Remove generated files"
	@echo ""
	@echo "Dependencies:"
	@echo "  make web         - Emscripten SDK"
	@echo "  make linux       - gcc, zlib, X11, OpenGL"
	@echo "  make macos       - brew install raylib"
	@echo "  make windows     - MinGW-w64 gcc (x86_64-w64-mingw32-gcc) and zlib"
	@echo "  make static      - same as linux"
	@echo ""
	@echo "Note: For WebAssembly builds, Lua must be compiled with Emscripten."
	@echo "      If you get linking errors, you may need to compile Lua with emcc."

.PHONY: web production linux macos windows static clean help
