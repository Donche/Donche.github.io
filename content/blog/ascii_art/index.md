---
layout: post
title: Create ASCII Art with Python
summary: Create ASCII Art with Python
header-style: text
description: "no"
category: python
date: "2019-01-05"
---

*It has been a long time since the last time I wrote a blog. This time I will be talking about a small project that can be used to generate ASCII Art from images or videos, and eventually play the generated ASCII video with music on. The source code can be found [here](https://github.com/Donche/ASCII_ART).*   

# 0. About ASCII art

This is surely something interesting to talk about. ASCII art is a graphic design technique that uses computers for presentation and consists of pictures pieced together from the 95 printable (from a total of 128) characters defined by the ASCII Standard from 1963 and ASCII compliant character sets with proprietary extended characters[[1]](#1). It is originally used because text is more stable and faster than images. However, because of the rapid development of technology, we don't really need ASCII art to be used to transfer image information any more. It has now become an interesting and somewhat retro artistic expression.   

For example, an image of ASCII art of Jackie Chan on the Internet is shown below. simple characters, amazing and yet expressive.   

![ASCII art of Jackie Chan](http://www.asciify.net/ascii/image/4672/jackie-chan.png)

More interestingly, ASCII art is actually very easy to generate in python. With only about 100 lines of code, we can generate and displaye the ASCII art. Here I'll show some code and explain how it works.   

# 1. Convert image to ASCII art

The main idea of this part is to map the gray value of each pixel in the image to the value corresponding to the dictionary we defined.   

We use PIL in python to process images. Firstly, use `open()` to open images. One thing important, for color images, regardless of whether the image format is PNG, BMP, or JPG, in PIL, after the `Open()` of the Image module is opened, the mode of the returned image object is "RGB". However we won't be needing the values of "RGB". So the function `convert()` is used to transform it to a gray image.   

After the transformation, we need to map the gray value to a certain character in `CODE_LIB`, which is the dictionary that we have previously decided to use in ASCII art. So suppose we have `n` characters in the dictionary, each character will correspond to `256/n` values. So we loop through each pixel in the image and store the converted result in memory. Here's the code:   

```python
CODE_LIB = r"B8&WM#YXQO{}[]()I1i!pao;:,.    "
count = len(CODE_LIB)
def transform_ascii(image_file): 
    image_file = image_file.convert("L") # convert image to gray
    code_pic = '' # result ASCII code
    for h in range(0,image_file.size[1]):
        for w in range(0,image_file.size[0]): 
            gray = image_file.getpixel((w,h))
            code_pic = code_pic + CODE_LIB[int(((count-1)*gray)/256)]
        code_pic = code_pic + "\n" 
    return code_pic
```

Then, if we display characters, we can find that because an ASCII character is actually much larger than a pixel, the resulting image is too large to be displayed in the console. Therefore, the image must be scaled to make the characters fit the screen.

So the function to convert image to an ASCII art and save it to `result.txt` is as follows:

```python
def convert_image():
    fn = input("input file name : ")
    hratio = float(input("input height zoom ratio(default 1.0) : ") or "1.0")
    wratio = float(input("input width zoom ratio(default 1.0) : ") or "1.0")
    image_file = Image.open(fn)
    image_file=image_file.resize((int(image_file.size[0]*wratio), int(image_file.size[1]*hratio)))
    print(u'Size info:',image_file.size[0],' ',image_file.size[1],' ')
    fo = open('result.txt','w')
    trans_data = transform_ascii(image_file)
    print(trans_data)
    fo.write(trans_data)
    fo.close()
```

For example, here's what it do to a random image of galaxy.



![ascii galaxy](img/ascii_galaxy.jpg)
![ascii galaxy result](img/ascii_galaxy_result.png)


It's definitely not the best result we can get, but, you've got the idea.

# 2. Convert video to ASCII art

Essentially, video is what many pictures displayed one after the other in the background sound. So, converting video to ASCII art is basically the same with what we did in the previous part, except that we have to process the pictures one by one. And for each frame of a video, we do exactly the same process and save the result in a file.   

To do so, we need to import OpenCV library, which is a famous cross-platform computer vision library. It's often used for image processing.    

The code to convert a video to a series of ASCII art files is as below:

```python
def convert_video():
    fn = input("input file name : ")
    hratio = float(input("input height zoom ratio(default 1.0) : ") or "1.0")
    wratio = float(input("input width zoom ratio(default 1.0) : ") or "1.0")
    cap = cv2.VideoCapture(fn) 
    i = 0
    if(os.path.isdir("./out") == False):
        os.makedirs("./out")
    while(cap.isOpened()): 
        ret, frame = cap.read() 
        if ret == False:
             break
        cv2.imshow('image', frame) 
        k = cv2.waitKey(5)

        os.system('cls') 
        i += 1
        tmp = open('./out/RES('+str(i)+').txt','w') 
        frame = cv2.resize(frame, (0,0), fx=wratio, fy=hratio)
        frame = Image.fromarray(cv2.cvtColor(frame,cv2.COLOR_BGR2RGB)) 
        trans_data = transform_ascii(frame) 
        print(trans_data) 
        tmp.write(trans_data) 
        tmp.close() 

        if (k & 0xff == ord('q')): 
            break 
    cap.release() 
    cv2.destroyAllWindows() 
```

# 3. Play ASCII art video

To generate ASCII art from a video, it's quite simple. But to play a ASCII art video, we will need more attention. The basic idea is to display the generated results one by one, with or without the music. However, the flash during two ASCII images is a big problem, and the synchronization with the sound is another. If we just start playing music and then do some normal display in terminal, there is a high probability that it will be a mess: the video is not fluid at all and the music is misaligned.

## 3.1 Screen painting

I use `curses` to deal with the refreshing problem. `curses` is a library who supplies a terminal-independent screen-painting and keyboard-handling facility for text-based terminals[[2]](#2). It provides lots of fancy skills to control the terminals: adding text, erasing it, changing its appearance, etc. I'll not give a tutorial on `curses` here, but the required functions will be introduced.

Before printing the ASCII images, `curses` must be initialized by the function `stdscr = curses.initscr()`. Here's what I did:

```python
stdscr = curses.initscr() # initialize the terminal with curses
stdscr.addstr(0,0,data) # add string to the terminal but will not be printed
stdscr.refresh() #refresh the terminal immediately
```

## 3.2 Synchronization

My original practice is to display all the results one by one and wait a certain amount of time to achieve the required playback speed. However, it took too much time to read the file and display it. Even if it didn't wait any time, it wouldn't catch up the play speed.

Therefore, I decided to ignore some frames during playback(Another possible solution is to read all files in advance). The number of the desired frame is calculated based on the time of the video, the number of frames and the current time. Suppose that the number of frames is `N`, the time of the video `T`, and the current time `t`, the number of the frame required is then: $t = (int)T/N*t+1$(frame number starts from 1). 

In this case, the frame won't be displayed one by one. Instead, for each frame, the number will be calculated precisely. The code is as follows:

```python
def play_ascii_video():
    v = float(input("Frame per second: \t"))
    pmusic = input("play music?(y/n)\t")
    if(pmusic == 'y'):
        filedir = input("input music name:\t")
        thread = Thread(target = play_music, args = (filedir,))
        thread.start()
    stdscr = curses.initscr()
    stdscr.keypad(1)
    t0 = time.time()
    i = 1
    while(True):
        t1 = time.time()
        i = int((t1 - t0) * v) + 1
        filename = './out/BA('+str(i)+').txt'
        try:
            with open(filename,'r') as f:
                data = f.read()
                stdscr.addstr(0,0,data)
                stdscr.addstr("Frame: %d"%i)
                stdscr.refresh()
        except IOError:
            break
```

# 4. Conclusion

Anyway, it's really a small project that can be done in one afternoon. But it still required more skills than I expected, especially when I found out that the video playing is not what I expected at all. I tried it with a famous video [BadApple](https://www.youtube.com/watch?v=FtutLA63Cp8)(I heard it from one of my roommates who is a truly crazy ACG fan...) and the result is satisfying.

Well, of course, in theory, this code can be used for any video conversion. It suits better when the video is in the form of silhouette, but we can still use any video we like and see what it give us. It's so much fun!

*Reference Materials*

1. <span id="1"></span> [ASCII art - Wikipedia](https://en.wikipedia.org/wiki/ASCII_art)    

2. <span id="1"></span>[Curses Programming with Python](https://docs.python.org/3/howto/curses.html)

