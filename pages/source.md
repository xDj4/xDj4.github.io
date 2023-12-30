asd

```c
/**
 * \file
 * \brief gfx_write Source File.
 *
 * \note
 *
 * Filename:    gfx_write.c
 * Created:        11/17/2016    16:56:45
 * Author:        David Johnstone
 * Version:        1.41a
 *
 * Copyright (C) 2012-2016 Cantrol Corporation. All Rights Reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted only for use by Cantrol Corporation,
 * its employees, factory authorized dealers, and any authorized contractors.
 *
 * Any other use is strictly prohibited.
 *
 * THIS SOFTWARE IS PROVIDED BY CANTROL "AS IS" AND ANY EXPRESS OR IMPLIED
 * WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF
 * MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NON-INFRINGEMENT ARE
 * EXPRESSLY AND SPECIFICALLY DISCLAIMED. IN NO EVENT SHALL CANTROL BE LIABLE FOR
 * ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
 * DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
 * OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
 * HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT,
 * STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN
 * ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
 * POSSIBILITY OF SUCH DAMAGE.
 *
 */

#include "gfx_include.h"

// Draw a character
void gfx_draw_char(int16_t x, int16_t y, unsigned char c, uint16_t color, uint16_t bg, uint8_t size)
{
    if(!GFX.TEXT.FONT) { // 'Classic' built-in font

        if((x >= GFX.SIZE.WIDTH)            || // Clip right
                (y >= GFX.SIZE.HEIGHT)           || // Clip bottom
                ((x + 6 * size - 1) < 0) || // Clip left
                ((y + 8 * size - 1) < 0))   // Clip top
            return;

        if(!GFX.CTRL.CP437 && (c >= 176)) c++; // Handle 'classic' charset behavior

        for(int8_t i = 0; i < 6; i++ ) {
            uint8_t line;

            if(i < 5) line = pgm_read_byte(FONT_DFLT + (c * 5) + i);
            else      line = 0x0;

            for(int8_t j = 0; j < 8; j++, line >>= 1) {
                if(line & 0x1) {
                    if(size == 1) gfx_draw_pixel(x + i, y + j, color);
                    else          gfx_fill_rect(x + (i * size), y + (j * size), size, size, color);
                }
                else if(bg != color) {
                    if(size == 1) gfx_draw_pixel(x + i, y + j, bg);
                    else          gfx_fill_rect(x + i * size, y + j * size, size, size, bg);
                }
            }
        }
    }

#ifdef CFG_GFX_CSTM_FONTS
    else { 
        c -= pgm_read_byte(&((GFXfont *)GFX.TEXT.FONT)->first);
        GFXglyph *glyph  = &(((GFXglyph *)pgm_read_ptr_near(&((GFXfont *)GFX.TEXT.FONT)->glyph))[c]);
        uint8_t  *bitmap = (uint8_t *)pgm_read_ptr_near(&((GFXfont *)GFX.TEXT.FONT)->bitmap);

        uint16_t bo = pgm_read_word(&glyph->bitmapOffset);
        uint8_t  w  = pgm_read_byte(&glyph->width),
                 h  = pgm_read_byte(&glyph->height),
                 xa = pgm_read_byte(&glyph->xAdvance);
        int8_t   xo = pgm_read_byte(&glyph->xOffset),
                 yo = pgm_read_byte(&glyph->yOffset);
        uint8_t  xx, yy, bits, bit = 0;
        int16_t  xo16, yo16;

        if(size > 1) {
            xo16 = xo;
            yo16 = yo;
        }

        for(yy = 0; yy < h; yy++) {
            for(xx = 0; xx < w; xx++) {
                if(!(bit++ & 7)) {
                    bits = pgm_read_byte(&bitmap[bo++]);
                }

                if(bits & 0x80) {
                    if(size == 1) {
                        gfx_draw_pixel(x + xo + xx, y + yo + yy, color);
                    }
                    else {
                        gfx_fill_rect(x + (xo16 + xx)*size, y + (yo16 + yy)*size, size, size, color);
                    }
                }

                bits <<= 1;
            }
        }
    } /* End classic vs custom font */

    /* CFG_GFX_CSTM_FONTS */
#endif
}

// Write a character
size_t gfx_write(uint16_t c)
{

#ifdef CFG_GFXM
    if (iscntrl(c)) {
        return 0;
    }
#endif

    if (c >= GFX_CNTRL_OFFSET) {
        c -= GFX_CNTRL_OFFSET;
    }

    if(!GFX.TEXT.FONT) { // 'Classic' built-in font

        if(c == '\n') {
            GFX.CURSOR.Y += GFX.TEXT.SIZE * 8;
            GFX.CURSOR.X  = 0;
        }
        else if(c == '\r') {
            // skip em
        }
        else {
            if(GFX.CTRL.WRAP && ((GFX.CURSOR.X + GFX.TEXT.SIZE * 6) >= GFX.SIZE.WIDTH)) { // Heading off edge?
                GFX.CURSOR.X  = 0;            // Reset x to zero
                GFX.CURSOR.Y += GFX.TEXT.SIZE * 8; // Advance y one line
            }

            gfx_draw_char(GFX.CURSOR.X, GFX.CURSOR.Y, c, GFX.TEXT.COLOR.FORE, GFX.TEXT.COLOR.BACK, GFX.TEXT.SIZE);
            GFX.CURSOR.X += GFX.TEXT.SIZE * 6;
        }
    }

#ifdef CFG_GFX_CSTM_FONTS
    else { // Custom font

        if(c == '\n') {
            GFX.CURSOR.X  = 0;
            GFX.CURSOR.Y += (int16_t)GFX.TEXT.SIZE *
                            (uint8_t)pgm_read_byte(&((GFXfont *)GFX.TEXT.FONT)->yAdvance);
        }
        else if(c != '\r') {
            uint8_t first = pgm_read_byte(&((GFXfont *)GFX.TEXT.FONT)->first);

            if((c >= first) && (c <= (uint8_t)pgm_read_byte(&((GFXfont *)GFX.TEXT.FONT)->last))) {
                uint8_t   c2    = c - pgm_read_byte(&((GFXfont *)GFX.TEXT.FONT)->first);
                GFXglyph *glyph = &(((GFXglyph *)pgm_read_ptr_near(&((GFXfont *)GFX.TEXT.FONT)->glyph))[c2]);
                uint8_t   w     = pgm_read_byte(&glyph->width),
                          h     = pgm_read_byte(&glyph->height);

                if((w > 0) && (h > 0)) { // Is there an associated bitmap?
                    int16_t xo = (int8_t)pgm_read_byte(&glyph->xOffset); // sic

                    if(GFX.CTRL.WRAP && ((GFX.CURSOR.X + GFX.TEXT.SIZE * (xo + w)) >= GFX.SIZE.WIDTH)) {
                        // Drawing character would go off right edge; wrap to new line
                        GFX.CURSOR.X  = 0;
                        GFX.CURSOR.Y += (int16_t)GFX.TEXT.SIZE *
                                        (uint8_t)pgm_read_byte(&((GFXfont *)GFX.TEXT.FONT)->yAdvance);
                    }

                    gfx_draw_char(GFX.CURSOR.X, GFX.CURSOR.Y, c, GFX.TEXT.COLOR.FORE, GFX.TEXT.COLOR.BACK, GFX.TEXT.SIZE);
                }

                GFX.CURSOR.X += pgm_read_byte(&glyph->xAdvance) * (int16_t)GFX.TEXT.SIZE;
            }
        }
    }

#endif // CFG_GFX_CSTM_FONTS
    return 1;
}
```
