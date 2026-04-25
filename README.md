import javax.swing.*;
import java.awt.*;
import java.awt.event.*;
import java.util.Random;
import com.fazecast.jSerialComm.SerialPort;
import java.util.Scanner;

/**
 * =============================================================================
 * EECS 1021 - Final Project: Hardware-Integrated Snake Game
 * Java Software Layer: Real-time Control via Raspberry Pi Pico 2W
 * =============================================================================
 *
 * BEHAVIOR (System Logic):
 * - Makes a UART serial connection with the Raspberry Pi Pico 2W
 * - Uses the Pico for analog data from the M5Stack Angle Unit to set direction.
 * - Maps the 10-bit analog to digital conversion values (0-1023)
 *   to Snake directions: UP, DOWN, LEFT, RIGHT.
 * - When an apple is "eaten", it then sends a 'B' (Beep) trigger byte back to the hardware.
 * - Handles hardware disconnection well by using a try catch exception handling.
 *
 * HARDWARE ASSUMPTIONS (Connections):
 * - Microcontroller: The Raspberry Pi Pico 2W is connected using USB (UART/Serial).
 * - Sensor: M5Stack Angle Unit (Potentiometer) Signal on GP26 (ADC0).
 * - Actuator: Active Buzzer (Adafruit 1536) Positive (+) on GP11.
 * - Visual Indicator: Onboard Pico LED used as another reward signal.
 *
 * PROGRAM FLOW (Integration Architecture):
 * 1. main() starts the SnakeGameHardware JFrame and starts the GUI.
 * 2. initSerial() will attempt to open the COM port connection and then initializes the UART connection.
 * 3. actionPerformed() runs as the game loop, fires every ~140ms (can be adjusted depending on feel of the game).
 * 4. readHardwareDirection() uses the serial input stream to update movement variables.
 * 5. checkApple() triggers the hardware actuator (Buzzer/LED) by writing to the serial port.
 *
 * @author Nathan Williamson, Chiron Hechanova, Jerry Giovannone, Seyed Ryan Hosseini
 */

public class SnakeGameHardware extends JPanel implements ActionListener {

    private final int SCREEN_WIDTH = 600;
    private final int SCREEN_HEIGHT = 600;
    private final int UNIT_SIZE = 25;
    private final int GAME_UNITS = (SCREEN_WIDTH * SCREEN_HEIGHT) / UNIT_SIZE;
    private int currentDelay = 140;
    private final int x[] = new int[GAME_UNITS];
    private final int y[] = new int[GAME_UNITS];
    private int bodyParts = 6;
    private int applesEaten, appleX, appleY;
    private char direction = 'R';
    private boolean running = false;
    private Timer timer;
    private Random random;
    private SerialPort sp;

    // This is what initializes the serial communication
    // This also sets up the dimensions game panel
    public SnakeGameHardware() {
        initSerial();
        random = new Random();
        this.setPreferredSize(new Dimension(SCREEN_WIDTH, SCREEN_HEIGHT));
        this.setBackground(Color.black);
        this.setFocusable(true);
        startGame();
    }

    // This attempts to open the UART port and uses a try catch to check if the device is missing
    private void initSerial() {
        try {
            sp = SerialPort.getCommPort("COM3");
            sp.setComPortParameters(115200, 8, 1, 0);
            sp.setComPortTimeouts(SerialPort.TIMEOUT_NONBLOCKING, 0, 0);
            if (!sp.openPort()) throw new Exception("Port failed to open");
        } catch (Exception e) {
            System.err.println("Hardware Error: Could not connect to Pico 2W.");
        }
    }

    // This resets the state of the game, spawns first apple, and starts a timer for the swing
    public void startGame() {
        newApple();
        running = true;
        timer = new Timer(currentDelay, this);
        timer.start();
    }

    // This will cancel the paint method, then redraw the screen on each frame
    public void paintComponent(Graphics g) {
        super.paintComponent(g);
        draw(g);
    }

    // Makes the red apple, green snake, and prints the current score
    public void draw(Graphics g) {
        if (running) {
            g.setColor(Color.red);
            g.fillOval(appleX, appleY, UNIT_SIZE, UNIT_SIZE);
            for (int i = 0; i < bodyParts; i++) {
                g.setColor(i == 0 ? Color.green : new Color(45, 180, 0));
                g.fillRect(x[i], y[i], UNIT_SIZE, UNIT_SIZE);
            }
            g.setColor(Color.white);
            g.setFont(new Font("SansSerif", Font.BOLD, 20));
            g.drawString("Score: " + applesEaten, 10, 30);
        } else gameOver(g);
    }

    // Creates the new apple and spawns it at a random point in the grid
    public void newApple() {
        appleX = random.nextInt((int) (SCREEN_WIDTH / UNIT_SIZE)) * UNIT_SIZE;
        appleY = random.nextInt((int) (SCREEN_HEIGHT / UNIT_SIZE)) * UNIT_SIZE;
    }

    // Reads the angle unit and will shift all the of the body to the position of the one ahead
    public void move() {
        readHardwareDirection();
        for (int i = bodyParts; i > 0; i--) {
            x[i] = x[i - 1];
            y[i] = y[i - 1];
        }
        if (direction == 'U') y[0] -= UNIT_SIZE;
        else if (direction == 'D') y[0] += UNIT_SIZE;
        else if (direction == 'L') x[0] -= UNIT_SIZE;
        else if (direction == 'R') x[0] += UNIT_SIZE;
    }

    // Safely converts the analog to digital output in a try catch to make sure that the connection doesn't break
    private void readHardwareDirection() {
        try {
            if (sp != null && sp.bytesAvailable() > 0) {
                Scanner s = new Scanner(sp.getInputStream());
                if (s.hasNextInt()) {
                    int val = s.nextInt();
                    if (val < 250) direction = 'U';
                    else if (val < 500) direction = 'R';
                    else if (val < 750) direction = 'D';
                    else direction = 'L';
                }
            }
        } catch (Exception e) {
            System.err.println("Communication Interrupted: Using last known direction.");
        }
    }

    // Finds when the head of the snake collides with an apple, then sends a signal to the buzzer to trigger
    public void checkApple() {
        /* Detects head-apple collision and sends a byte to trigger the buzzer reward. */
        if ((x[0] == appleX) && (y[0] == appleY)) {
            bodyParts++;
            applesEaten++;
            if (sp != null && sp.isOpen()) {
                sp.writeBytes(new byte[]{'B'}, 1);
            }
            newApple();
        }
    }

    // Checks if the head collides with either the boundary or its own body
    public void checkCollisions() {
        for (int i = bodyParts; i > 0; i--) {
            if ((x[0] == x[i]) && (y[0] == y[i])) running = false;
        }
        if (x[0] < 0 || x[0] >= SCREEN_WIDTH || y[0] < 0 || y[0] >= SCREEN_HEIGHT) running = false;
        if (!running) timer.stop();
    }

    // Once games over, displays final score and game over on the screen.
    public void gameOver(Graphics g) {
        g.setColor(Color.red);
        g.setFont(new Font("SansSerif", Font.BOLD, 75));
        g.drawString("Game Over", 100, SCREEN_HEIGHT / 2);
    }

    // Runs the game logic loop and repaints the screen
    @Override
    public void actionPerformed(ActionEvent e) {
        if (running) {
            move();
            checkApple();
            checkCollisions();
        }
        repaint();
    }

    // Creates the main window and adds the hardware controlled graphic user interface (GUI)
    public static void main(String[] args) {
        JFrame frame = new JFrame("Final Project: Snake Game");
        frame.add(new SnakeGameHardware());
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.pack();
        frame.setVisible(true);
        frame.setLocationRelativeTo(null);
    }
}
