# Third-party code

## MAME (BSD-3-Clause)

The machine model in `core/missile.c` — memory map and its mirrors, the MADSEL
video path including `vram_mad_r`, `vram_mad_w` and the third-bit address
scramble, the /32V interrupt chain, the half-speed CPU over the bottom 32 lines,
and the screen layout — is written from MAME's `src/mame/atari/missile.cpp`.
`core/pokey.c` is written from `src/devices/sound/pokey.cpp`. Copyright Aaron
Giles, Nicola Salmoria and the MAME team; used under the BSD-3-Clause license:

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

1. Redistributions of source code must retain the above copyright notice, this
   list of conditions and the following disclaimer.
2. Redistributions in binary form must reproduce the above copyright notice,
   this list of conditions and the following disclaimer in the documentation
   and/or other materials provided with the distribution.
3. Neither the name of the copyright holder nor the names of its contributors
   may be used to endorse or promote products derived from this software
   without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT,
STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY
WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF
SUCH DAMAGE.

## The 6502

`core/m6502fast.h` was written for this project and is covered by the project
license. It passes Klaus Dormann's 6502 functional test in full, including
decimal mode.

## ROMs

No game ROMs are included in this repository, and none ever will be. Supplying
them is up to whoever builds the thing.
