import threading
from functools import partial
import cv2
from kivy.app import App
from kivy.clock import Clock
from kivy.graphics.texture import Texture
from kivy.lang import Builder
from kivy.uix.screenmanager import ScreenManager, Screen
import numpy as np
from object_detector import *
class MainScreen(Screen):
    pass
class Manager(ScreenManager):
    pass
Builder.load_string('''
<firstScreen>:

<MainScreen>:
    name: "Test"
    FloatLayout:
        Label:
            text: "Webcam from OpenCV?"
            pos_hint: {"x":0.0, "y":0.8}
            size_hint: 1.0, 0.2
        Image:
            # this is where the video will show
            # the id allows easy access
            id: vid
            size_hint: 1, 0.6
            allow_stretch: True  # allow the video image to be scaled
            keep_ratio: True  # keep the aspect ratio so people don't look squashed
            pos_hint: {'center_x':0.5, 'top':0.8}
        Button:
            text: 'Stop Video'
            pos_hint: {"x":0.0, "y":0.0}
            size_hint: 1.0, 0.2
            font_size: 50
            on_release: app.stop_vid()
''')
class Main(App):
    def build(self):
        # start the camera access code on a separate thread
        # if this was done on the main thread, GUI would stop
        # daemon=True means kill this thread when app stops
        threading.Thread(target=self.doit, daemon=True).start()
        sm = ScreenManager()
        self.main_screen = MainScreen()
        sm.add_widget(self.main_screen)
        return sm
    def doit(self):
        parameters = cv2.aruco.DetectorParameters_create()
        aruco_dict = cv2.aruco.Dictionary_get(cv2.aruco.DICT_5X5_50)


# Load Object Detector
        detector = HomogeneousBgDetector()

        # this code is run in a separate thread
        self.do_vid = True  # flag to stop loop
        # make a window for use by cv2
        # flags allow resizing without regard to aspect ratio
        cam = cv2.VideoCapture(0)
        # start processing loop
        while (self.do_vid):
            ret, frame = cam.read()

    # Get Aruco marker
            corners, _, _ = cv2.aruco.detectMarkers(frame, aruco_dict, parameters=parameters)
            if corners:

                # Draw polygon around the marker
                int_corners = np.int0(corners)
                cv2.polylines(frame, int_corners, True, (0, 255, 0), 5)

                # Aruco Perimeter
                aruco_perimeter = cv2.arcLength(corners[0], True)

                # Pixel to cm ratio
                pixel_cm_ratio = aruco_perimeter / 20

                contours = detector.detect_objects(frame)

                # Draw objects boundaries
                for cnt in contours:
                    # Get rect
                    rect = cv2.minAreaRect(cnt)
                    (x, y), (w, h), angle = rect

                    # Get Width and Height of the Objects by applying the Ratio pixel to cm
                    object_width = w / pixel_cm_ratio
                    object_height = h / pixel_cm_ratio

                    # Display rectangle
                    box = cv2.boxPoints(rect)
                    box = np.int0(box)

                    cv2.circle(frame, (int(x), int(y)), 5, (0, 0, 255), -1)
                    cv2.polylines(frame, [box], True, (255, 0, 0), 2)
                    cv2.putText(frame, "Width {} cm".format(round(object_width, 1)), (int(x - 100), int(y - 20)), cv2.FONT_HERSHEY_PLAIN, 2, (100, 200, 0), 2)
                    cv2.putText(frame, "Height {} cm".format(round(object_height, 1)), (int(x - 100), int(y + 15)), cv2.FONT_HERSHEY_PLAIN, 2, (100, 200, 0), 2)



          
            # ...

            # more code
            # ...
            # send this frame to the kivy Image Widget
            # Must use Clock.schedule_once to get this bit of code
            # to run back on the main thread (required for GUI operations)
            # the partial function just says to call the specified method with the provided argument (Clock adds a time argument)
            Clock.schedule_once(partial(self.display_frame, frame))
        cam.release()
        cv2.destroyAllWindows()
    def stop_vid(self):
        # stop the video capture loop
        self.do_vid = False
    def display_frame(self, frame, dt):
        # display the cur
        # rent video frame in the kivy Image widget
        # create a Texture the correct size and format for the frame
        texture = Texture.create(size=(frame.shape[1], frame.shape[0]), colorfmt='bgr')
        # copy the frame data into the texture
        texture.blit_buffer(frame.tobytes(order=None), colorfmt='bgr', bufferfmt='ubyte')
        # flip the texture (otherwise the video is upside down
        texture.flip_vertical()
        # actually put the texture in the kivy Image widget
        self.main_screen.ids.vid.texture = texture
if __name__ == '__main__':
    Main().run()










