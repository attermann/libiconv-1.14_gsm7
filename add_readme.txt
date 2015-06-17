Adding libiconv Support for the IMAP Variant of UTF-7

By Weldon Whipple <weldon@whipple.org>

Introduction

The IMAP Variant of UTF-7 is used to encode non-ASCII characters in mail folder names. This allows any character in the Unicode (UCS-2) character set to appear in a mail folder name.

The original UTF-7 encoding is documented in "RFC 1642 - UTF-7 - A Mail-Safe Transformation Format of Unicode." The variant for the encoding of mail folders is documented in section 5.1.3 of "RFC 2060 - Internet Message Access Protocol - Version 4rev1."

Most IMAP-capable e-mail clients (Mozilla, Netscape and Outlook, for example) support the UTF-7 IMAP encoding.

You can download the latest libiconv distribution from http://www.gnu.org/software/libiconv/.

Build Instructions

To add UTF-7 IMAP support to libiconv:

If you haven't already, download a copy of the libiconv tarball and unzip/tar the contents.
You will need the gperf program to regenerate some hash files after adding UTF-7 IMAP support. Type the command "which gperf" to determine if it is installed on your build machine. If it isn't, download the source and build and install gperf.
(I searched Google for a copy, downloaded it and built and installed it on a box that was lacking ...)
Copy the file utf7imap.h to the ./lib subdirectory of the libiconv distribution.
Edit the file ./lib/encodings.def and add the following:
DEFENCODING(( "UTF-7-IMAP",             /* RFC 2060, sec. 5.1.3 */
              "UTF7IMAP",               
            ),
            utf7imap,
            { utf7imap_mbtowc, NULL },     { utf7imap_wctomb,
              utf7imap_reset })
I inserted it immediately after a block for "standard" UTF-7.
Edit the file ./lib/converters.h and insert the following line:
#include "utf7imap.h"
I inserted it immediately after the line for utf7.h
Edit the file ./lib/aliases.gperf and insert the following lines:
UTF-7-IMAP, ei_utf7imap
UTF7IMAP, ei_utf7imap
I inserted them right after the line that reads:
		     
   CSUNICODE11UTF7, ei_utf7
Regenerate the ./lib/aliases.h file by issuing the following command (from the base libiconv directory):
% make -f Makefile.devel lib/aliases.h
Regenerate the ./lib/flags.h file by issuing the following command:
% make -f Makefile.devel lib/flags.h
Finally, rebuild libiconv:
% make clean; make; make install
If this is a brand new build of libiconv--and you haven't previously configured, built and installed libiconv--you might be able to just follow the libiconv instructions for configuring, building, checking and installing libiconv, instead of this last step.
Idiosyncrasies

RFC 2060 documents the differences between standard UTF-7 and the IMAP variant used for naming mailboxes. Two differences are probably worth repeating here:

In standard UTF-7, printable US-ASCII characters (bytes in the range 0x20-0x25 and 0x27-0x7e can either represent themselves or be encoded in base64. In UTF-7 IMAP, on the other hand, "modified Base64 MUST NOT be used to represent any printing US-ASCII character which can represent itself."
In other "stateful" encodings--as well as in UTF-7 IMAP--strings begin in the US-ASCII state. In those same encodings (NOT including UTF-7 IMAP), changing state back to US-ASCII is optional at the end of the string or line: end of string or line automatically resets the state back to the US-ASCII state.
The end-of-string in UTF-7 IMAP (on the other hand) must include bytes that set the encoding back to the US-ASCII state!

Modifying Text::Iconv

If you use Perl's Text::Iconv, you might need to modify the C coded Iconv.xs file to satisfy the requirement that all UTF-7 IMAP strings terminate in the US-ASCII state.

The do_conv function of the Iconv.xs file has the following while loop, which converts a buffer from Unicode (UCS-2) to some other encoding:

   while(inbytesleft != 0)
   {
#ifdef __hpux
      /* Even in HP-UX 11.00, documentation and header files do not
   agree */
      ret = iconv(iconv_handle, &icursor, &inbytesleft,
                                &ocursor, &outbytesleft);
#else
      ret = iconv(iconv_handle, (const char **)&icursor, &inbytesleft,
                                &ocursor, &outbytesleft);
#endif

      if(ret == (size_t) -1)
      {
         switch(errno)
         {
            case EILSEQ:
               /* Stop conversion if input character encountered which
                  does not belong to the input char set */
               if (raise_error)
                  croak("Character not from source char set: %s",
                        strerror(errno));
               Safefree(obuf);
               return(&PL_sv_undef);
            case EINVAL:
               /* Stop conversion if we encounter an incomplete
                  character or shift sequence */
               if (raise_error)
                  croak("Incomplete character or shift sequence: %s",
                        strerror(errno));
               Safefree(obuf);
               return(&PL_sv_undef);
            case E2BIG:
               /* If the output buffer is not large enough, copy the
                  converted bytes to the return string, reset the
                  output buffer and continue */
               sv_catpvn(perl_str, obuf, l_obuf - outbytesleft);
               ocursor = obuf;
               outbytesleft = l_obuf;
               break;
            default:
               if (raise_error)
                  croak("iconv error: %s", strerror(errno));
               Safefree(obuf);
               return(&PL_sv_undef);
         }
      }
   }
If the target encoding is UTF-7 IMAP, the above loop will not--for certain strings--end the UTF-7 IMAP string in the US-ASCII state. So, following the above loop, we need to insert one more call to iconv, passing a NULL value for the input buffer. (This causes iconv to output the final hyphen '-' to return to US-ASCII state). The following code is a copy of the "innerds" of the above while loop, with the second argument to iconv changed to NULL. The code is inserted immediately after the above while loop:

#if 1
   /*  Addition so that (if the output
        encoding is stateful), trailing bytes will be added to
        to the output string to return the string to its initial
        shift state. This is critical for some encodings, such
        as the IMAP variant of UTF-7. When converting from UCS-2 to
        UTF-7-IMAP, if the last character is a non-ASCII character
        we need to output the final '-' to return the string to
        the initial ASCII state. */

#ifdef __hpux
      /* Even in HP-UX 11.00, documentation and header files do not
   agree */
      ret = iconv(iconv_handle, NULL, &inbytesleft,
                                &ocursor, &outbytesleft);
#else
      ret = iconv(iconv_handle, (const char **)NULL, &inbytesleft,
                                &ocursor, &outbytesleft);
#endif
      if(ret == (size_t) -1)
      {
         switch(errno)
         {
            case EILSEQ:
               /* Stop conversion if input character encountered which
                  does not belong to the input char set */
               if (raise_error)
                  croak("Character not from source char set: %s",
                        strerror(errno));
               Safefree(obuf);
               return(&PL_sv_undef);
            case EINVAL:
               /* Stop conversion if we encounter an incomplete
                  character or shift sequence */
               if (raise_error)
                  croak("Incomplete character or shift sequence: %s",
                        strerror(errno));
               Safefree(obuf);
               return(&PL_sv_undef);
            case E2BIG:
               /* If the output buffer is not large enough, copy the
                  converted bytes to the return string, reset the
                  output buffer and continue */
               sv_catpvn(perl_str, obuf, l_obuf - outbytesleft);
               ocursor = obuf;
               outbytesleft = l_obuf;
               break;
            default:
               if (raise_error)
                  croak("iconv error: %s", strerror(errno));
               Safefree(obuf);
               return(&PL_sv_undef);
         }
      }
#endif
One Final Modification to Iconv.xs

You will need increase the outbytesleft temporary output buffer just ahead of the while loop above. (The additional call to iconv might add characters for a partially converted base64 character, plus the trailing '-'.) We need at least two more bytes. (I increase by 4 for good measure.) If you don't increase the output buffer length, certain inputs will cause buffer overflow. If the output buffer is too small, libiconv will not crash--it will just truncate the final characters added by the code we inserted above.

Find the lines (just before the while loop in do_conv above) that read:

   if(inbytesleft <= MB_LEN_MAX)
   {
      outbytesleft = MB_LEN_MAX + 1;
   }
   else
   {
      outbytesleft = 2 * inbytesleft;
   }
Immediately following, add this:

   /* We need a larger output buffer to hold possible extra bytes 
      for ending the conversion for utf-7-imap. This
      number is arbitrary and may need to be larger... */
   outbytesleft += 4;
Then rebuild Text::Iconv.

Errors, Suggestions?

Please report errors in this documentation or send suggestions to Weldon Whipple <weldon@whipple.org>.
