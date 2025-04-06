---
layout: default
title: 2-key Morse pad
nav_order: 3
---
# CH32_Morse_keyboard

Link to git repo: [https://github.com/snoopy3921/CH32_Morse_keyboard](https://github.com/snoopy3921/CH32_Morse_keyboard)

This board is CH32x035 Weact and it uses Ch32x035 microcontroller with USB support and bootloader with boot button. Use WCHISPStudio tool to upload hex file to the board.

I'm using USB library and example from WCH. You can find out [here](https://github.com/openwch/ch32x035)

To send a key to PC, USB device will send 8 bytes a time. First byte is to notify about shift, ctrl,... Second byte is reserved, key will be sent from second byte. API to send to PC:

```c
USBFS_Endp_DataUp( DEF_UEP1, KB_Data_Pack, sizeof( KB_Data_Pack ), DEF_UEP_CPY_LOAD );
```

For morsing, i create a morse object, modify macros to change its behavior.

```c
#define MORSE_BUFFER_LEN 10
#define MORSE_TIMEOUT_TO_SEND 1800
#define MORSE_TIMEOUT_MAX 5000
typedef struct
{
    char buffer[MORSE_BUFFER_LEN];
    char char_to_send;
    uint8_t index;
    uint8_t is_buffer_empty;
    uint32_t timeout;
}morse_t;

extern morse_t my_morse;
```

And also the key code map

```c
typedef struct {
    const char *morse;
    char letter;
} MorseCode;

#define KEY_SPACE 0x2c // Keyboard Spacebar
#define KEY_A 0x04 // Keyboard a and A
#define KEY_B 0x05 // Keyboard b and B
#define KEY_C 0x06 // Keyboard c and C
#define KEY_D 0x07 // Keyboard d and D
#define KEY_E 0x08 // Keyboard e and E
#define KEY_F 0x09 // Keyboard f and F
#define KEY_G 0x0a // Keyboard g and G
#define KEY_H 0x0b // Keyboard h and H
#define KEY_I 0x0c // Keyboard i and I
#define KEY_J 0x0d // Keyboard j and J
#define KEY_K 0x0e // Keyboard k and K
#define KEY_L 0x0f // Keyboard l and L
#define KEY_M 0x10 // Keyboard m and M
#define KEY_N 0x11 // Keyboard n and N
#define KEY_O 0x12 // Keyboard o and O
#define KEY_P 0x13 // Keyboard p and P
#define KEY_Q 0x14 // Keyboard q and Q
#define KEY_R 0x15 // Keyboard r and R
#define KEY_S 0x16 // Keyboard s and S
#define KEY_T 0x17 // Keyboard t and T
#define KEY_U 0x18 // Keyboard u and U
#define KEY_V 0x19 // Keyboard v and V
#define KEY_W 0x1a // Keyboard w and W
#define KEY_X 0x1b // Keyboard x and X
#define KEY_Y 0x1c // Keyboard y and Y
#define KEY_Z 0x1d // Keyboard z and Z

#define KEY_1 0x1e // Keyboard 1 and !
#define KEY_2 0x1f // Keyboard 2 and @
#define KEY_3 0x20 // Keyboard 3 and #
#define KEY_4 0x21 // Keyboard 4 and $
#define KEY_5 0x22 // Keyboard 5 and %
#define KEY_6 0x23 // Keyboard 6 and ^
#define KEY_7 0x24 // Keyboard 7 and &
#define KEY_8 0x25 // Keyboard 8 and *
#define KEY_9 0x26 // Keyboard 9 and (
#define KEY_0 0x27 // Keyboard 0 and )

MorseCode morse_dict[] = {
    {".-", KEY_A}, {"-...", KEY_B}, {"-.-.", KEY_C}, {"-..", KEY_D},
    {".", KEY_E}, {"..-.", KEY_F}, {"--.", KEY_G}, {"....", KEY_H},
    {"..", KEY_I}, {".---", KEY_J}, {"-.-", KEY_K}, {".-..", KEY_L},
    {"--", KEY_M}, {"-.", KEY_N}, {"---", KEY_O}, {".--.", KEY_P},
    {"--.-", KEY_Q}, {".-.", KEY_R}, {"...", KEY_S}, {"-", KEY_T},
    {"..-", KEY_U}, {"...-", KEY_V}, {".--", KEY_W}, {"-..-", KEY_X},
    {"-.--", KEY_Y}, {"--..", KEY_Z},
    {"-----", KEY_0}, {".----", KEY_1}, {"..---", KEY_2}, {"...--", KEY_3},
    {"....-", KEY_4}, {".....", KEY_5}, {"-....", KEY_6}, {"--...", KEY_7},
    {"---..", KEY_8}, {"----.", KEY_9},
    {"------", KEY_SPACE} // Space for word separation
};
```

The most fun part seems to be designing the case for it. 
<center>
<img src="/assets/images/Top_view.png"/>
</center>
<center>
<img src="/assets/images/Bot_view.png"/>
</center>
<center>
<img src="/assets/images/Real.jpg"/>
</center>

With the boot button at the bottom, we can reprogram it via bootloader.

Demo
<iframe width="480" height="720" src="https://www.youtube.com/embed/wLfgBXRENvs" frameborder="0" allow="autoplay" allowfullscreen></iframe>

25/03/2025 