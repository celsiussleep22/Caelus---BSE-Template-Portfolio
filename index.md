# Thermal Imaging Camera
My project was the thermal imaging camera, it uses the mlx90640 thermal camera to get the information, then uses the pi to convert that information Replace this text with a brief description (2-3 sentences) of your project. This description should draw the reader in and make them interested in what you've built. You can include what the biggest challenges, takeaways, and triumphs from completing the project were. As you complete your portfolio, remember your audience is less familiar than you are with all that your project entails!

<!--You should comment out all portions of your portfolio that you have not completed yet, as well as any instructions: -->


| **Engineer** | **School** | **Area of Interest** | **Grade** |
|:--:|:--:|:--:|:--:|
| Caelus B | Los Altos High School | Electrical Engineering | Incoming Sophmore

<!--**Replace the BlueStamp logo below with an image of yourself and your completed project. Follow the guide [here](https://tomcam.github.io/least-github-pages/adding-images-github-pages-site.html) if you need help.** -->

![Headstone Image](logo.svg)



# First Milestone


<iframe width="1004" height="565" src="https://www.youtube.com/embed/82A-ptBor8o" title="Caelus B.  Milestone 1" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

-- My project used the raspberry pi connected to a thermal camera. I coded the pi to have a bar to show the colors for each temrpeature and had a display of the image from the camera. Currently, i'm struggling with debugging my code, it doesn't work. My plan is to add a portable screen and battery to it, and give it the ability to take pictures


# Final Milestone


<iframe width="869" height="589" src="https://www.youtube.com/embed/hNtC5qa7hIc?list=PLe-u_DjFx7etvdoxgh04tIDzwMjn92btk" title="Caelus B. Milestone 2" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

-- My final milestone was to add modifications. I made the whole thing portable, with its own screen and battery. I also slightly increase the qulity of the image. My biggest challenge in this proejct was the code that didn't work from the website, I eventually overcame it by doing research and asking for help. They key topics I learned were overall engineering from the kahoots, and the coding portion of my project. Something I hope to learn in the future after everything i learned here is some more mechanical enginering, this porject was mostly code, so I want to learn about more of the physical things

# Schematics 
<img width="664" height="974" alt="image" src="https://github.com/user-attachments/assets/d4045d9d-5cb8-4226-b141-db6909d6406b" />


# Code

```python
import time
import board
import busio
import numpy as np
import adafruit_mlx90640
import cv2
from collections import deque

def initialize_sensor():
    i2c = busio.I2C(board.SCL, board.SDA)
    mlx = adafruit_mlx90640.MLX90640(i2c)
    mlx.refresh_rate = adafruit_mlx90640.RefreshRate.REFRESH_16_HZ
    return mlx

UPSCALE = 8
BUFFER_SIZE = 8
frame_buffer = deque(maxlen=BUFFER_SIZE)

def super_resolve(buffer, scale=UPSCALE):
    """
    Average recent frames then upscale — reduces noise and recovers
    sub-pixel detail that shifts between reads due to sensor noise.
    """
    stacked = np.mean(buffer, axis=0)
    h, w = stacked.shape
    upscaled = cv2.resize(stacked, (w * scale, h * scale),
                          interpolation=cv2.INTER_LANCZOS4)
    blurred = cv2.GaussianBlur(upscaled, (0, 0), sigmaX=2)
    sharpened = cv2.addWeighted(upscaled, 1.8, blurred, -0.8, 0)
    return sharpened

def normalize_and_colormap(data):
    mn, mx = np.min(data), np.max(data)
    norm = ((data - mn) / (mx - mn + 1e-6) * 255).astype(np.uint8)
    return cv2.applyColorMap(norm, cv2.COLORMAP_INFERNO)

def add_overlay(frame_bgr, data_array, fps):
    mn, mx = np.min(data_array), np.max(data_array)
    center = data_array[12, 16]
    for i, text in enumerate([
        f'Min: {mn:.1f}C',
        f'Max: {mx:.1f}C',
        f'Ctr: {center:.1f}C',
        f'FPS: {fps:.1f}'
    ]):
        cv2.putText(frame_bgr, text, (10, 25 + i * 25),
                    cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 255, 255), 1)

def main():
    mlx = initialize_sensor()
    frame = np.zeros((24 * 32,))
    t_array = []
    cv2.namedWindow('Thermal Camera', cv2.WINDOW_NORMAL)
    cv2.resizeWindow('Thermal Camera', 640, 480)
    print("Press 'q' to quit")
    while True:
        t1 = time.monotonic()
        try:
            mlx.getFrame(frame)
        except (ValueError, RuntimeError):
            continue
        data_array = np.fliplr(np.reshape(frame, (24, 32)))
        frame_buffer.append(data_array.copy())
        if len(frame_buffer) < 3:
            continue
        resolved = super_resolve(frame_buffer)
        bgr = normalize_and_colormap(resolved)
        t_array.append(time.monotonic() - t1)
        fps = len(t_array) / np.sum(t_array)
        add_overlay(bgr, data_array, fps)
        cv2.imshow('Thermal Camera', bgr)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break
    cv2.destroyAllWindows()

if __name__ == '__main__':
    main()
```

```python
import time
import board
import busio
import numpy as np
import adafruit_mlx90640
import cv2

i2c = busio.I2C(board.SCL, board.SDA)
mlx = adafruit_mlx90640.MLX90640(i2c)
mlx.refresh_rate = adafruit_mlx90640.RefreshRate.REFRESH_16_HZ

cv2.namedWindow('Thermal Camera', cv2.WINDOW_NORMAL)
cv2.resizeWindow('Thermal Camera', 640, 520)

frame = np.zeros((24 * 32,))
t_array = []
max_retries = 5

def draw_colorbar(image, min_temp, max_temp):
    """Draws a colorbar strip at the bottom of the image"""
    h, w = image.shape[:2]
    bar_height = 40
    bar = np.linspace(0, 255, w, dtype=np.uint8).reshape(1, -1)
    bar = cv2.applyColorMap(np.tile(bar, (bar_height, 1)), cv2.COLORMAP_INFERNO)
    cv2.putText(bar, f'{min_temp:.1f}C', (5, 28),
                cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 255, 255), 1)
    cv2.putText(bar, f'{max_temp:.1f}C', (w - 70, 28),
                cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 255, 255), 1)
    cv2.putText(bar, 'Temperature [C]', (w//2 - 70, 28),
                cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 255, 255), 1)
    return np.vstack([image, bar])

print("Press 'q' to quit")
while True:
    t1 = time.monotonic()
    retry_count = 0
    while retry_count < max_retries:
        try:
            mlx.getFrame(frame)
            data_array = np.fliplr(np.reshape(frame, (24, 32)))
            mn, mx = np.min(data_array), np.max(data_array)
            norm = ((data_array - mn) / (mx - mn + 1e-6) * 255).astype(np.uint8)
            upscaled = cv2.resize(norm, (640, 480), interpolation=cv2.INTER_CUBIC)
            colored = cv2.applyColorMap(upscaled, cv2.COLORMAP_INFERNO)
            display = draw_colorbar(colored, mn, mx)
            cv2.imshow('Thermal Camera', display)
            t_array.append(time.monotonic() - t1)
            fps = len(t_array) / np.sum(t_array)
            print(f'Sample Rate: {fps:2.1f}fps')
            break
        except ValueError:
            retry_count += 1
        except RuntimeError as e:
            retry_count += 1
            if retry_count >= max_retries:
                print(f"Failed after {max_retries} retries with error: {e}")
                break
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break
cv2.destroyAllWindows()
```

# Bill of Materials
Here's where you'll list the parts in your project. To add more rows, just copy and paste the example rows below.
Don't forget to place the link of where to buy each component inside the quotation marks in the corresponding row after href =. Follow the guide [here]([url](https://www.markdownguide.org/extended-syntax/)) to learn how to customize this to your project needs. 

| **Part** | **Note** | **Price** | **Link** |
|:--:|:--:|:--:|:--:|
| Hosyond Screen | Used for displaying the image | $45.99 | <a href="https://www.amazon.com/dp/B09XKC53NH?lv=shuf&social_share=cm_sw_r_cp_ud_dp_4P1369E4GD7MTFKSAQQV_1&channelId=751&ref_=cm_sw_r_cp_ud_dp_4P1369E4GD7MTFKSAQQV_1&plpRedirect=mhFallback&th=1"> Link </a> |
| Power Bank | Used to make the whole thing portable by supplying power | $25.99 | <a href="https://www.amazon.com/Anker-Travel-Ready-Technology-High-Speed-Output（Black），1pack/dp/B0D5CLSMFB/ref=sr_1_10?crid=3SV5HEYTS5WJX&dib=eyJ2IjoiMSJ9.bSWFqslAPpD-Wi_zW-SHpTgGirvs3z9FUgRi4c2uf1d5pfIVn_gLcuLDYU_PnXRiaBGBNiU2CRdwqoxXZfZI_uxJaR1N00jGsxCbOuAQ0SNvfFxL6GYX8WLdAy4s4fj2zFsu072WK4YWcf1DvbH9yeLLpCBVlMjGkCDEozAM3ELoJXRwbYmZaGOLr5Hqu0AA91k0jrlK7OQMhrVPCnls8zaiuYfszql313tSVvNE_Hk.ZPep1j_qow8FwLoSuXcn0MneJ99BHGcSlG5kP4vQfIw&dib_tag=se&keywords=5v%2Bpowerbank&qid=1781636747&sprefix=5v%2Bpowerbank%2Caps%2C196&sr=8-10&th=1"> Link </a> |
| Raspberry Pi 4 | Used to convert the information fromt the camera into an image | $148.99 | <a href="https://www.amazon.com/dp/B0C8LV6VNZ?ref=cm_sw_r_cp_ud_dp_ZSXTKYVG1X4S0JWKJMXB&ref_=cm_sw_r_cp_ud_dp_ZSXTKYVG1X4S0JWKJMXB&social_share=cm_sw_r_cp_ud_dp_ZSXTKYVG1X4S0JWKJMXB&th=1"> Link </a> |
| MLX90640| Used as a camera to show  | $148.99 | <a href="https://www.amazon.com/dp/B0C8LV6VNZ?ref=cm_sw_r_cp_ud_dp_ZSXTKYVG1X4S0JWKJMXB&ref_=cm_sw_r_cp_ud_dp_ZSXTKYVG1X4S0JWKJMXB&social_share=cm_sw_r_cp_ud_dp_ZSXTKYVG1X4S0JWKJMXB&th=1"> Link </a> |


<!--
# Other Resources/Examples
One of the best parts about Github is that you can view how other people set up their own work. Here are some past BSE portfolios that are awesome examples. You can view how they set up their portfolio, and you can view their index.md files to understand how they implemented different portfolio components.
- [Example 1](https://trashytuber.github.io/YimingJiaBlueStamp/)
- [Example 2](https://sviatil0.github.io/Sviatoslav_BSE/)
- [Example 3](https://arneshkumar.github.io/arneshbluestamp/)

To watch the BSE tutorial on how to create a portfolio, click here.
-->
