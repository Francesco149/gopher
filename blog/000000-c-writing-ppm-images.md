old writeup from my 2016 gopherspace, slightly revised and simplified

ppm is the simplest image format that has decent support in image viewers. a ppm file can be written
in 10 lines of code with no libraries and piped directly into a viewer, which is extremely neat
for things like writing a tiny raytracer

ppm starts with a header plain text string that contains:
* a magic string "P6"
* width, height in pixels
* max color value (<= 0xffff)

each element of the header is separated by whitespace (either spaces, tab, CR, LF) and is terminated
by a whitespace character

for example the header for a 640x480 image with 255 colors would be

    P6 640 480 255


the header is followed by the binary pixel data.

the order of the pixel data is as straightforward as it gets: rows are written from top to bottom,
each row contains r, g, b values from left to right.

r, g, b values are 1 byte each if the maximum color value is less or equal than 0xff, otherwise
2 bytes

so, if we make a 2x2 checkerboard of white and black pixels in a 255 colors image, the data will
look like this:

    ff ff ff 00 00 00 00 00 00 ff ff ff

now that we know the ppm format, we can easily produce said image and display it:

    $ printf "P6 2 2 255
        \xff\xff\xff\x00\x00\x00\x00\x00\x00\xff\xff\xff" > test.ppm

    $ od -Ax -w10 -tx1z test.ppm
    000000 50 36 20 32 20 32 20 32 35 35  >P6 2 2 255<
    00000a 0a ff ff ff 00 00 00 00 00 00  >..........<
    000014 ff ff ff                       >...<
    000017

    $ display -sample 50x50 test.ppm

of course, we can pipe the file directly into display:

    $ printf "P6 2 2 255
        \xff\xff\xff\x00\x00\x00\x00\x00\x00\xff\xff\xff" \
        | display -sample 50x50 -

note that the "sample 50x50 -" parameters are purely to make this 2x2px image visible since it's so
small, with a normal picture just piping into "display" will be enough.

now that we have mastered the ppm format, writing a c function to produce a ppm file is trivial

I strongly recommend you go and try writing the program yourself, it's a good excercise and it's
more fun than getting spoonfed

if you aren't a complete beginner at C you can jump to /p4 to see the full code which should be self
explanatory

our function will take:
* file stream to write the image to
* array of pixels as float rgb values between 0.0f and 1.0f
* width and height

we will only use 255 colors to avoid having to specify the color depth every time.

    void FPutPPM(FILE* f, float* px, int width, int height) {
        /* ??? */
    }

Writing the header is a simple fprintf (include stdio.h):

    fprintf(f, "P6 %u %u 255\n", width, height);

we need to convert loop over the array and convert float color values to bytes. remember, we have
an array of floats with 3 values for each pixels, so we need to iterate `width * height * 3` items

note that this assumes that your color values are already clamped between 0-1

    for (i = 0; i < width * height * 3; ++i) {
        fputc((unsigned char)(px[i] * 255), f);
    }

and just like that, we have written the pixel data

here's an example program that generates a checkerboard and writes it to stdout

    /* gcc ppm.c && ./a.out | display */

    #include <stdio.h>

    void FPutPPM(FILE* f, float* px, int width, int height) {
      int i;
      fprintf(f, "P6 %d %d 255\n", width, height);
      for (i = 0; i < width * height * 3; ++i) {
        fputc((unsigned char)(px[i] * 255), f);
      }
    }

    #define NCELLS 8
    #define CELL_W 32
    #define W (NCELLS * CELL_W)

    int main(int argc, char* argv[]) {
        float px[W * W * 3];
        float* p = px;
        int i, j;
        for (j = 0; j < W; ++j) {
          for (i = 0; i < W; ++i) {
            int xodd = (i / CELL_W) & 1;
            int yodd = (j / CELL_W) & 1;
            p[0] = p[2] = 1.f; /* magenta */
            if (xodd ^ yodd) {
              p[1] = 1.f;
            }
            p += 3;
          }
        }
        FPutPPM(stdout, px, W, W);
        return 0;
    }

