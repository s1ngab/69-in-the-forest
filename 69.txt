import javax.swing.*;
import java.awt.*;
import java.awt.event.*;
import java.awt.geom.Ellipse2D;
import java.util.ArrayList;
import java.util.HashSet;
import java.util.Random;
import java.util.Set;

public class Main {
    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            JFrame window = new JFrame(ForestGame.GAME_NAME);
            window.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
            window.setContentPane(new ForestGame());
            window.pack();
            window.setLocationRelativeTo(null);
            window.setVisible(true);
        });
    }
}

class ForestGame extends JPanel {
    static final String GAME_NAME = "내 방식";
    static final String PLAYER_NAME = "솔";
    static final int WORLD = 1800, CAMP = WORLD / 2;
    static final double DAY_SECONDS = 45, NIGHT_SECONDS = 30;
    static final double ZOOM = 1.75;
    static final double SWING_DURATION = 0.5;
    static final double SCYTHE_REACH = 85;
    int swingFacing = 1;
    double walkTime;
    int playerFacing = 1;
    final Random random = new Random();
    final Set<Integer> keys = new HashSet<>();
    final ArrayList<Point> trees = new ArrayList<>();
    final ArrayList<Point.Double> enemies = new ArrayList<>();
    final ArrayList<Bunny> bunnies = new ArrayList<>();
    Bunny interactingBunny;
    boolean bunnyPetted;
    int cutscenePage;
    double cutsceneTime;
    boolean snackQuestAccepted, questAnnouncementOpen, questMenuOpen;
    // Quest names stay English-only, including titles added for future quests.
    static final String SNACK_QUEST_TITLE = "snack for Conejito";
    static final String[] BUNNY_STORY = {
            "솔: \"토끼다!\"",
            "코네히토: \"안녕.\"",
            "솔: \"잘 지내?\"",
            "코네히토: \"먹을 것 좀 줘.\"",
            "솔: \"어... 나 먹을 게 없는데.\"",
            "코네히토: \"먹을 것 좀 달라고 했잖아. (  •̀ ᴖ •́  )\""
    };
    static final String[] BUNNY_STORY_ENGLISH = {
            "Sol: \"A bunny!\"",
            "Conejito: \"Hola.\"",
            "Sol: \"What's up?\"",
            "Conejito: \"Give me a snack.\"",
            "Sol: \"Uh... I don't have any food.\"",
            "Conejito: \"I told you to give me a snack. (  •̀ ᴖ •́  )\""
    };
    static final double BUNNY_INTERACTION_RANGE = 75;
    Bunny promptBunny;
    double promptOpacity, questFadeTime;

    static class Bunny {
        double x, y, angle, hopTime, restTime;
        int facing = 1;
        boolean special;

        Bunny(double x, double y, double restTime) {
            this.x = x; this.y = y; this.restTime = restTime;
        }
    }
    double x, y, health, fuel, phaseTime, spawnTime, attackCooldown, hitFlash;
    int wood, night;
    boolean darkness, started, ended;
    String message = "";
    double messageTime;
    long lastTick = System.nanoTime();

    ForestGame() {
        setPreferredSize(new Dimension(1000, 720));
        setBackground(new Color(20, 37, 30));
        reset();
        bind(KeyEvent.VK_W); bind(KeyEvent.VK_A); bind(KeyEvent.VK_S); bind(KeyEvent.VK_D);
        bind(KeyEvent.VK_UP); bind(KeyEvent.VK_LEFT); bind(KeyEvent.VK_DOWN); bind(KeyEvent.VK_RIGHT);
        bind(KeyEvent.VK_E); bind(KeyEvent.VK_SPACE); bind(KeyEvent.VK_ENTER); bind(KeyEvent.VK_R);
        bind(KeyEvent.VK_F); bind(KeyEvent.VK_ESCAPE);
        bind(KeyEvent.VK_Q);
        addHierarchyListener(e -> {
            if ((e.getChangeFlags() & HierarchyEvent.DISPLAYABILITY_CHANGED) != 0 && isDisplayable()) {
                SwingUtilities.getWindowAncestor(this).addWindowFocusListener(new WindowAdapter() {
                    @Override public void windowLostFocus(WindowEvent event) { keys.clear(); }
                });
            }
        });
        new Timer(16, e -> {
            long now = System.nanoTime();
            double dt = Math.min((now - lastTick) / 1e9, 0.05);
            lastTick = now;
            if (started && !ended) updateGame(dt);
            if (interactingBunny != null && interactingBunny.special) cutsceneTime += dt;
            updateUi(dt);
            repaint();
        }).start();
    }

    void bind(int key) {
        String name = Integer.toString(key);
        getInputMap(WHEN_IN_FOCUSED_WINDOW).put(KeyStroke.getKeyStroke(key, 0, false), name);
        getInputMap(WHEN_IN_FOCUSED_WINDOW).put(KeyStroke.getKeyStroke(key, 0, true), name + "Up");
        getActionMap().put(name, new AbstractAction() {
            public void actionPerformed(ActionEvent e) {
                if (!keys.add(key)) return;
                if (questAnnouncementOpen || questMenuOpen) {
                    if (key == KeyEvent.VK_Q) {
                        questMenuOpen = questAnnouncementOpen || !questMenuOpen;
                        questAnnouncementOpen = false;
                        keys.clear();
                    } else if (key == KeyEvent.VK_ESCAPE || key == KeyEvent.VK_ENTER) {
                        questAnnouncementOpen = false;
                        questMenuOpen = false;
                        keys.clear();
                    }
                    return;
                }
                if (interactingBunny != null) {
                    if (interactingBunny.special) {
                        if (key == KeyEvent.VK_ESCAPE) {
                            interactingBunny = null;
                            keys.clear();
                        } else if (key == KeyEvent.VK_ENTER || key == KeyEvent.VK_F || key == KeyEvent.VK_SPACE) {
                            if (cutsceneTime * 35 < BUNNY_STORY[cutscenePage].length()) {
                                cutsceneTime = BUNNY_STORY[cutscenePage].length() / 35.0 + 1;
                            } else if (++cutscenePage >= BUNNY_STORY.length) {
                                interactingBunny = null;
                                keys.clear();
                                if (!snackQuestAccepted) {
                                    snackQuestAccepted = true;
                                    questAnnouncementOpen = true;
                                    questFadeTime = 0;
                                }
                            } else cutsceneTime = 0;
                        }
                        return;
                    }
                    if (key == KeyEvent.VK_F || key == KeyEvent.VK_ESCAPE) {
                        interactingBunny = null;
                        keys.clear();
                    } else if (key == KeyEvent.VK_E) {
                        bunnyPetted = true;
                    }
                    return;
                }
                if (key == KeyEvent.VK_ENTER && !started) started = true;
                if (key == KeyEvent.VK_R && ended) { reset(); started = true; }
                if (started && !ended && key == KeyEvent.VK_Q) {
                    questMenuOpen = true;
                    keys.clear();
                    return;
                }
                if (started && !ended && key == KeyEvent.VK_F) {
                    interactingBunny = nearestBunny();
                    bunnyPetted = false;
                    cutscenePage = snackQuestAccepted ? BUNNY_STORY.length - 1 : 0;
                    cutsceneTime = 0;
                    if (interactingBunny == null) say("얼룩 토끼에게 가까이 가서 말을 걸어 보세요.\nMove closer to the spotted bunny to interact.");
                }
                if (started && !ended && key == KeyEvent.VK_E) interact();
                if (started && !ended && key == KeyEvent.VK_SPACE) attack();
            }
        });
        getActionMap().put(name + "Up", new AbstractAction() {
            public void actionPerformed(ActionEvent e) { keys.remove(key); }
        });
    }

    void reset() {
        x = CAMP; y = CAMP + 90; health = 100; fuel = 75; wood = 3; night = 1;
        phaseTime = 0; spawnTime = 0; attackCooldown = 0; hitFlash = 0;
        darkness = false; ended = false;
        walkTime = 0; playerFacing = 1;
        swingFacing = 1;
        interactingBunny = null; bunnyPetted = false;
        cutscenePage = 0; cutsceneTime = 0;
        snackQuestAccepted = false; questAnnouncementOpen = false; questMenuOpen = false;
        promptBunny = null; promptOpacity = 0; questFadeTime = 0;
        trees.clear(); enemies.clear(); keys.clear();
        bunnies.clear();
        for (int i = 0; i < 18; i++) {
            // A few start near the clearing so they are easy to discover.
            double angle = random.nextDouble() * Math.PI * 2;
            double radius = i < 5 ? 190 + random.nextDouble() * 110 : 250 + random.nextDouble() * 550;
            bunnies.add(new Bunny(CAMP + Math.cos(angle) * radius,
                    CAMP + Math.sin(angle) * radius, random.nextDouble() * 2));
        }
        Bunny guardian = new Bunny(CAMP + 210, CAMP + 130, 2);
        guardian.special = true;
        bunnies.add(guardian);
        for (int i = 0; i < 150; i++) addTree();
        say("해가 지기 전에 나무를 모으고 모닥불을 지키세요!\nGather wood before sunset. Keep the campfire burning!");
    }

    void addTree() {
        Point tree;
        do { tree = new Point(55 + random.nextInt(WORLD - 110), 55 + random.nextInt(WORLD - 110)); }
        while (tree.distance(CAMP, CAMP) < 170);
        trees.add(tree);
    }

    void say(String text) { message = text; messageTime = 4; }
    boolean down(int a, int b) { return keys.contains(a) || keys.contains(b); }

    void updateGame(double dt) {
        // Pause the forest while the interaction menu is open.
        if (interactingBunny != null || questAnnouncementOpen || questMenuOpen) return;
        double dx = (down(KeyEvent.VK_D, KeyEvent.VK_RIGHT) ? 1 : 0) - (down(KeyEvent.VK_A, KeyEvent.VK_LEFT) ? 1 : 0);
        double dy = (down(KeyEvent.VK_S, KeyEvent.VK_DOWN) ? 1 : 0) - (down(KeyEvent.VK_W, KeyEvent.VK_UP) ? 1 : 0);
        double length = Math.hypot(dx, dy);
        if (length > 0) {
            walkTime += dt * 11;
            if (dx != 0) playerFacing = dx < 0 ? -1 : 1;
            x = Math.clamp(x + dx / length * 190 * dt, 20, WORLD - 20);
            y = Math.clamp(y + dy / length * 190 * dt, 20, WORLD - 20);
        } else walkTime = 0;
        attackCooldown = Math.max(0, attackCooldown - dt);
        // The blade connects during the sweep, after the initial wind-up.
        if (attackCooldown > 0.08 && attackCooldown <= 0.38)
            enemies.removeIf(enemy -> enemy.distance(x, y) < SCYTHE_REACH);
        hitFlash = Math.max(0, hitFlash - dt);
        messageTime -= dt;
        updateBunnies(dt);
        fuel = Math.max(0, fuel - dt * (darkness ? 1.05 : 0.3));
        double campDistance = Math.hypot(x - CAMP, y - CAMP);
        if (fuel > 0 && campDistance < 145) health = Math.min(100, health + dt * 2);
        phaseTime += dt;
        if (phaseTime >= (darkness ? NIGHT_SECONDS : DAY_SECONDS)) {
            phaseTime = 0;
            if (darkness) {
                darkness = false; enemies.clear();
                night++;
                for (int i = 0; i < 18 && trees.size() < 150; i++) addTree();
                say("날이 밝았습니다! 나무를 모아 " + night + "번째 밤을 준비하세요.\nDawn! Gather wood for night " + night + ".");
            } else {
                darkness = true; spawnTime = 0;
                say("밤이 찾아왔습니다. 모닥불이 괴물들을 막아 줍니다.\nNight falls. The fire keeps creatures away.");
            }
        }
        if (darkness) {
            spawnTime -= dt;
            if (spawnTime <= 0) {
                double angle = random.nextDouble() * Math.PI * 2;
                enemies.add(new Point.Double(Math.clamp(x + Math.cos(angle) * 480, 10, WORLD - 10),
                        Math.clamp(y + Math.sin(angle) * 480, 10, WORLD - 10)));
                spawnTime = Math.max(1.2, 4 - night * 0.45);
            }
        }
        for (Point.Double enemy : enemies) {
            double ex = x - enemy.x, ey = y - enemy.y;
            double distance = Math.hypot(ex, ey);
            if (distance > 1) {
                double speed = 65 + night * 9;
                enemy.x += ex / distance * speed * dt;
                enemy.y += ey / distance * speed * dt;
            }
            double fromFire = enemy.distance(CAMP, CAMP);
            if (fuel > 0 && fromFire < 155) {
                double angle = Math.atan2(enemy.y - CAMP, enemy.x - CAMP);
                enemy.x = CAMP + Math.cos(angle) * 155;
                enemy.y = CAMP + Math.sin(angle) * 155;
            }
            if (enemy.distance(x, y) < 27) { health = Math.max(0, health - 22 * dt); hitFlash = 0.15; }
        }
        if (health <= 0) ended = true;
    }

    void updateUi(double dt) {
        if (questAnnouncementOpen) questFadeTime = Math.min(0.6, questFadeTime + dt);
        if (!started || ended || interactingBunny != null || questAnnouncementOpen || questMenuOpen) {
            promptBunny = null;
            promptOpacity = 0;
            return;
        }
        Bunny nearby = nearestBunny();
        if (nearby != null) {
            if (nearby != promptBunny) promptOpacity = 0;
            promptBunny = nearby;
            promptOpacity = Math.min(1, promptOpacity + dt / 0.25);
        } else {
            promptOpacity = Math.max(0, promptOpacity - dt / 0.18);
            if (promptOpacity == 0) promptBunny = null;
        }
    }

    void drawBunnyPrompt(Graphics2D graphics, double cameraX, double cameraY) {
        if (promptBunny == null || promptOpacity <= 0 || interactingBunny != null
                || questAnnouncementOpen || questMenuOpen || !started || ended) return;
        Graphics2D g = (Graphics2D) graphics.create();
        g.setComposite(AlphaComposite.SrcOver.derive((float)promptOpacity));
        int centerX = (int)((promptBunny.x - cameraX) * ZOOM);
        double hop = Math.sin(promptBunny.hopTime / 0.4 * Math.PI) * 9;
        double bunnyScale = promptBunny.special ? 1.8 : 1;
        int top = (int)((promptBunny.y - (33 + hop) * bunnyScale - cameraY) * ZOOM) - 60;
        g.setColor(new Color(12, 20, 22, 225));
        g.fillRoundRect(centerX - 98, top, 196, 50, 12, 12);
        String[] lines = {"F: 코네히토와 대화", "F: talk to Conejito"};
        for (int i = 0; i < lines.length; i++) {
            g.setFont(new Font("Dialog", i == 0 ? Font.BOLD : Font.PLAIN, i == 0 ? 15 : 11));
            g.setColor(i == 0 ? new Color(249, 206, 133) : new Color(207, 207, 215));
            g.drawString(lines[i], centerX - g.getFontMetrics().stringWidth(lines[i]) / 2, top + 21 + i * 17);
        }
        g.dispose();
    }

    Bunny nearestBunny() {
        Bunny nearest = null;
        double distance = BUNNY_INTERACTION_RANGE;
        for (Bunny bunny : bunnies) {
            if (!bunny.special) continue;
            double candidateDistance = Math.hypot(bunny.x - x, bunny.y - y);
            if (candidateDistance <= distance) {
                nearest = bunny;
                distance = candidateDistance;
            }
        }
        return nearest;
    }

    void updateBunnies(double dt) {
        for (Bunny bunny : bunnies) {
            // The guardian waits for visitors instead of fleeing.
            if (bunny.special && Math.hypot(bunny.x - x, bunny.y - y) < 170) {
                bunny.hopTime = 0;
                bunny.facing = x < bunny.x ? -1 : 1;
                continue;
            }
            boolean startled = !bunny.special && Math.hypot(bunny.x - x, bunny.y - y) < 100;
            if (bunny.hopTime <= 0) {
                bunny.restTime -= dt;
                if (startled || bunny.restTime <= 0) {
                    bunny.angle = startled ? Math.atan2(bunny.y - y, bunny.x - x)
                            : random.nextDouble() * Math.PI * 2;
                    if (bunny.special && Math.hypot(bunny.x - (CAMP + 210), bunny.y - (CAMP + 130)) > 100)
                        bunny.angle = Math.atan2(CAMP + 130 - bunny.y, CAMP + 210 - bunny.x);
                    // Turn back into the forest at its edges.
                    if (bunny.x < 55 || bunny.x > WORLD - 55 || bunny.y < 55 || bunny.y > WORLD - 55)
                        bunny.angle = Math.atan2(CAMP - bunny.y, CAMP - bunny.x);
                    bunny.facing = Math.cos(bunny.angle) < 0 ? -1 : 1;
                    bunny.hopTime = 0.4;
                    bunny.restTime = 0.6 + random.nextDouble() * 1.8;
                }
            }
            if (bunny.hopTime > 0) {
                double step = Math.min(dt, bunny.hopTime);
                double speed = bunny.special ? 35 : startled ? 120 : 60;
                bunny.x = Math.clamp(bunny.x + Math.cos(bunny.angle) * speed * step, 25, WORLD - 25);
                bunny.y = Math.clamp(bunny.y + Math.sin(bunny.angle) * speed * step, 25, WORLD - 25);
                bunny.hopTime = Math.max(0, bunny.hopTime - dt);
            }
        }
    }

    void drawBunny(Graphics2D graphics, Bunny bunny) {
        Graphics2D g = (Graphics2D) graphics.create();
        g.translate(bunny.x, bunny.y);
        if (bunny.special) g.scale(1.8, 1.8);
        g.setColor(new Color(10, 23, 16, 90));
        g.fillOval(-13, 4, 26, 9);
        double lift = Math.sin(bunny.hopTime / 0.4 * Math.PI) * 9;
        g.translate(0, -lift);
        g.scale(bunny.facing, 1);
        g.setColor(bunny.special ? new Color(249, 248, 245) : new Color(232, 224, 205));
        g.fillOval(-13, -9, 25, 18);
        g.fillOval(3, -16, 15, 16);
        g.fillOval(4, -33, 6, 22);
        g.fillOval(12, -31, 6, 20);
        if (bunny.special) {
            g.setColor(new Color(28, 29, 36));
            g.fillOval(-11, -7, 10, 10);
            g.fillOval(0, -4, 7, 8);
            g.fillOval(6, -16, 7, 7);
            g.fillOval(12, -31, 6, 10);
        }
        g.setColor(new Color(226, 163, 163));
        g.fillOval(6, -29, 2, 14);
        g.fillOval(14, -27, 2, 12);
        g.fillOval(16, -8, 4, 3);
        g.setColor(new Color(32, 36, 33));
        g.fillOval(12, -12, 3, 3);
        g.setColor(new Color(251, 247, 234));
        g.fillOval(-18, -6, 9, 9);
        g.fillOval(-8, 5, 10, 5);
        g.fillOval(6, 4, 9, 5);
        g.dispose();
    }

    void drawBunnyCutscene(Graphics2D graphics) {
        Graphics2D g = (Graphics2D) graphics.create();
        int w = getWidth(), h = getHeight();
        g.setPaint(new GradientPaint(0, 0, new Color(11, 17, 34), w, h, new Color(44, 30, 65)));
        g.fillRect(0, 0, w, h);
        g.setPaint(new RadialGradientPaint(w / 2f, h * 0.45f, Math.max(1, w * 0.4f),
                new float[]{0, 1}, new Color[]{new Color(125, 168, 232, 65), new Color(70, 70, 120, 0)}));
        g.fillRect(0, 0, w, h);
        for (int i = 0; i < 24; i++) {
            int px = (i * 137 + 43) % Math.max(1, w);
            int py = (int)(h * 0.18 + (i * 53 % Math.max(1, (int)(h * 0.4))));
            g.setColor(new Color(183, 214, 255, 50 + (int)(35 * (1 + Math.sin(cutsceneTime * 2 + i)))));
            g.fillOval(px, py, 3, 3);
        }
        Graphics2D portrait = (Graphics2D) g.create();
        portrait.translate(w / 2.0, h * 0.51 + Math.sin(cutsceneTime * 2) * 2);
        portrait.scale(3.5, 3.5);
        Bunny guardian = new Bunny(0, 0, 0); guardian.special = true;
        drawBunny(portrait, guardian);
        portrait.dispose();
        g.setColor(new Color(5, 9, 17));
        g.fillRect(0, 0, w, 55); g.fillRect(0, h - 48, w, 48);
        centered(g, "코네히토\nConejito", 28, 18, new Color(205, 211, 239));
        int boxY = (int)(h * 0.64);
        g.setColor(new Color(8, 14, 26, 230)); g.fillRoundRect(40, boxY, w - 80, h - boxY - 65, 18, 18);
        String line = BUNNY_STORY[cutscenePage];
        int visible = Math.min(line.length(), (int)(cutsceneTime * 35));
        int fontSize = Math.max(12, Math.min(21, (w - 120) / 35));
        centered(g, line.substring(0, visible), boxY + 55, fontSize, new Color(234, 233, 246));
        centered(g, BUNNY_STORY_ENGLISH[cutscenePage], boxY + 86,
                Math.max(10, fontSize - 6), new Color(172, 191, 219));
        centered(g, "Enter / F / Space: 계속     Esc: 나가기\nEnter / F / Space: continue     Esc: leave", h - 88, 14, new Color(172, 191, 219));
        centered(g, (snackQuestAccepted ? "1 / 1" : (cutscenePage + 1) + " / " + BUNNY_STORY.length)
                + "   •   일시 정지\nGame paused", h - 27, 12, new Color(172, 191, 219));
        g.dispose();
    }

    void interact() {
        if (Math.hypot(x - CAMP, y - CAMP) < 105) {
            if (wood == 0) say("나무가 필요합니다. 나무 근처에서 E를 누르세요.\nYou need wood. Walk near a tree and press E.");
            else if (fuel > 80) say("아직 모닥불의 연료가 충분합니다.\nThe fire has plenty of fuel for now.");
            else { wood--; fuel = Math.min(100, fuel + 25); say("모닥불에 나무를 넣었습니다.\nAdded wood to the campfire."); }
            return;
        }
        Point nearest = null;
        double distance = 78;
        for (Point tree : trees) {
            if (tree.distance(x, y) < distance) { nearest = tree; distance = tree.distance(x, y); }
        }
        if (nearest != null) { trees.remove(nearest); wood += 2; say("나무 2개를 모았습니다.\nCollected 2 wood."); }
        else say("나무에 가까이 가거나 모닥불로 돌아가세요.\nMove closer to a tree, or return to the campfire.");
    }

    void attack() {
        if (attackCooldown > 0) return;
        attackCooldown = SWING_DURATION;
        swingFacing = playerFacing;
    }

    void drawScythe(Graphics2D graphics) {
        Graphics2D g = (Graphics2D) graphics.create();
        g.translate(x, y);
        g.scale(attackCooldown > 0 ? swingFacing : playerFacing, 1);
        double progress = attackCooldown > 0 ? 1 - attackCooldown / SWING_DURATION : 0;
        // Ease through a full circular sweep and settle back into the idle pose.
        double sweep = progress * progress * (3 - 2 * progress);
        if (attackCooldown > 0.08 && attackCooldown < 0.4) {
            g.setStroke(new BasicStroke(4, BasicStroke.CAP_ROUND, BasicStroke.JOIN_ROUND));
            for (int i = 0; i < 5; i++) {
                g.setColor(new Color(156, 181, 255, 100 - i * 18));
                g.draw(new java.awt.geom.Arc2D.Double(-76, -76, 152, 152,
                        65 - sweep * 360 + i * 14, 14, java.awt.geom.Arc2D.OPEN));
            }
        }
        g.translate(13, 1);
        g.rotate(Math.toRadians(-12 + sweep * 360));
        // Reverse the resting blade while keeping the grip in the same hand.
        g.scale(-1, 1);
        g.setStroke(new BasicStroke(5, BasicStroke.CAP_ROUND, BasicStroke.JOIN_ROUND));
        g.setColor(new Color(55, 49, 44)); g.drawLine(0, 20, 0, -49);
        g.setStroke(new BasicStroke(2));
        g.setColor(new Color(128, 118, 104)); g.drawLine(-1, 19, -1, -49);
        // A curved, tapered blade in warm steel tones.
        java.awt.geom.Path2D blade = new java.awt.geom.Path2D.Double();
        blade.moveTo(-3, -48);
        blade.curveTo(16, -63, 43, -49, 48, -20);
        blade.curveTo(35, -40, 18, -44, 1, -39);
        blade.closePath();
        g.setPaint(new GradientPaint(0, -58, new Color(232, 226, 214),
                39, -24, new Color(137, 130, 119)));
        g.fill(blade);
        g.setColor(new Color(241, 233, 216)); g.setStroke(new BasicStroke(1.3f)); g.draw(blade);
        g.setColor(new Color(159, 142, 116)); g.fillOval(-4, -48, 8, 8);
        g.setColor(new Color(70, 57, 47));
        for (int i = -7; i <= 8; i += 4) g.drawLine(-3, i, 3, i + 2);
        g.setColor(new Color(237, 197, 160)); g.fillOval(-4, -3, 8, 8);
        g.dispose();
    }

    void drawPlayer(Graphics2D graphics) {
        Graphics2D g = (Graphics2D) graphics.create();
        g.translate(x, y);
        g.setColor(new Color(8, 17, 18, 95));
        g.fillOval(-17, 10, 34, 12);
        double step = Math.sin(walkTime) * 3;
        g.setColor(new Color(204, 207, 211));
        g.fillRoundRect(-10, 4 + (int)step, 8, 13, 4, 4);
        g.fillRoundRect(2, 4 - (int)step, 8, 13, 4, 4);
        g.setColor(new Color(69, 49, 43));
        g.fillRoundRect(-11, 13 + (int)step, 10, 6, 3, 3);
        g.fillRoundRect(1, 13 - (int)step, 10, 6, 3, 3);
        g.translate(0, -Math.abs(step) * 0.4);
        g.scale(playerFacing, 1);
        // Simple blocks of color keep the silhouette readable at gameplay scale.
        g.setColor(new Color(217, 222, 226));
        g.fillRoundRect(-15, -11, 8, 22, 7, 7);
        g.fillRoundRect(7, -11, 8, 22, 7, 7);
        g.setColor(new Color(246, 245, 239));
        g.fillRoundRect(-10, -14, 20, 28, 8, 8);
        g.setColor(new Color(222, 225, 225));
        g.fillRoundRect(-10, 8, 20, 6, 3, 3);
        g.setColor(new Color(232, 194, 163));
        g.fillOval(-15, 6, 6, 6); g.fillOval(9, 6, 6, 6);
        // One tidy shoulder strap and a compact pouch.
        g.setStroke(new BasicStroke(2, BasicStroke.CAP_ROUND, BasicStroke.JOIN_ROUND));
        g.setColor(new Color(83, 65, 63)); g.drawLine(-7, -10, 9, 7);
        g.fillRoundRect(6, 4, 9, 9, 3, 3);
        g.setColor(new Color(151, 118, 89)); g.fillRoundRect(6, 4, 9, 3, 2, 2);
        // Clip the plaid weave to the scarf's collar and hanging tail.
        java.awt.geom.Area scarf = new java.awt.geom.Area(
                new java.awt.geom.RoundRectangle2D.Double(-9, -15, 18, 6, 4, 4));
        scarf.add(new java.awt.geom.Area(
                new java.awt.geom.RoundRectangle2D.Double(-7, -10, 5, 10, 2, 2)));
        Graphics2D fabric = (Graphics2D) g.create();
        fabric.clip(scarf);
        fabric.setColor(new Color(186, 48, 54)); fabric.fill(scarf);
        fabric.setColor(new Color(64, 24, 34, 170));
        for (int stripe = -10; stripe < 10; stripe += 6) fabric.fillRect(stripe, -16, 2, 18);
        for (int stripe = -14; stripe < 1; stripe += 5) fabric.fillRect(-10, stripe, 20, 2);
        fabric.setStroke(new BasicStroke(0.6f));
        fabric.setColor(new Color(246, 163, 147, 180));
        fabric.drawLine(-1, -16, -1, 1);
        fabric.drawLine(-10, -11, 10, -11);
        fabric.drawLine(-10, -1, 10, -1);
        fabric.dispose();
        // Rounded face framed by fluffy, layered locks.
        g.setColor(new Color(232, 194, 163));
        g.fillOval(-12, -24, 4, 6); g.fillOval(8, -24, 4, 6);
        g.fillRoundRect(-10, -30, 20, 20, 12, 12);
        java.awt.geom.Path2D hair = new java.awt.geom.Path2D.Double();
        hair.moveTo(-11, -19);
        hair.curveTo(-17, -27, -14, -38, -5, -39);
        // One soft crown tuft above an otherwise rounded silhouette.
        hair.quadTo(-2, -40, 0, -42);
        hair.quadTo(2, -42, 2, -39);
        hair.curveTo(13, -40, 18, -28, 10, -19);
        hair.quadTo(11, -25, 7, -28);
        hair.quadTo(6, -25, 2, -23);
        hair.lineTo(3, -28);
        hair.quadTo(-1, -23, -5, -24);
        hair.lineTo(-4, -28);
        hair.quadTo(-9, -26, -11, -19);
        hair.closePath();
        g.setPaint(new GradientPaint(0, -42, new Color(30, 26, 29),
                0, -20, new Color(133, 85, 57)));
        g.fill(hair);
        g.setColor(new Color(99, 70, 57));
        g.setStroke(new BasicStroke(1.2f, BasicStroke.CAP_ROUND, BasicStroke.JOIN_ROUND));
        g.drawArc(-10, -36, 13, 9, 45, 100);
        g.drawArc(0, -36, 10, 8, 25, 105);
        g.setColor(new Color(37, 40, 52));
        g.fillOval(-5, -22, 3, 5);
        g.fillOval(3, -22, 3, 5);
        g.setColor(new Color(248, 245, 224));
        g.fillRect(-4, -22, 1, 1); g.fillRect(4, -22, 1, 1);
        g.setColor(new Color(171, 124, 111)); g.drawLine(0, -14, 2, -14);
        if (hitFlash > 0) {
            g.setColor(new Color(255, 92, 103, 100)); g.fillRoundRect(-16, -35, 33, 51, 15, 15);
        }
        g.dispose();
    }

    @Override protected void paintComponent(Graphics graphics) {
        super.paintComponent(graphics);
        Graphics2D g = (Graphics2D) graphics.create();
        g.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);
        java.awt.geom.AffineTransform screenTransform = g.getTransform();
        double cameraX = x - getWidth() / (2.0 * ZOOM), cameraY = y - getHeight() / (2.0 * ZOOM);
        g.scale(ZOOM, ZOOM);
        g.translate(-cameraX, -cameraY);
        g.setColor(new Color(33, 58, 41)); g.fillRect(0, 0, WORLD, WORLD);
        g.setColor(new Color(44, 70, 47));
        for (int i = 30; i < WORLD; i += 65)
            for (int j = 30; j < WORLD; j += 65) g.drawLine(i, j, i + 3, j - 6);
        g.setColor(new Color(89, 76, 51)); g.fillOval(CAMP - 115, CAMP - 115, 230, 230);
        if (fuel > 0) {
            g.setPaint(new RadialGradientPaint(CAMP, CAMP, 155,
                    new float[]{0f, 0.45f, 1f},
                    new Color[]{new Color(130, 220, 255, 80),
                            new Color(151, 105, 239, 45), new Color(115, 65, 210, 0)}));
            g.fillOval(CAMP - 155, CAMP - 155, 310, 310);
        }
        for (Point tree : trees) {
            g.setColor(new Color(17, 29, 24, 100)); g.fillOval(tree.x - 22, tree.y + 10, 53, 22);
            g.setColor(new Color(103, 73, 46)); g.fillRect(tree.x - 5, tree.y - 4, 10, 24);
            g.setColor(new Color(25, 82, 53));
            g.fillPolygon(new int[]{tree.x - 28, tree.x, tree.x + 28}, new int[]{tree.y + 5, tree.y - 53, tree.y + 5}, 3);
            g.setColor(new Color(40, 108, 65));
            g.fillPolygon(new int[]{tree.x - 21, tree.x, tree.x + 21}, new int[]{tree.y - 13, tree.y - 62, tree.y - 13}, 3);
        }
        g.setColor(new Color(132, 131, 119));
        for (int i = 0; i < 8; i++) {
            double angle = i * Math.PI / 4;
            g.fillOval(CAMP + (int)(Math.cos(angle) * 27) - 7, CAMP + (int)(Math.sin(angle) * 22) - 6, 14, 12);
        }
        g.setColor(new Color(94, 54, 30)); g.fillRoundRect(CAMP - 23, CAMP - 5, 46, 12, 5, 5);
        if (fuel > 0) {
            int flicker = (int)(Math.sin(System.nanoTime() / 1e8) * 4);
            g.setPaint(new LinearGradientPaint(CAMP, CAMP + 5, CAMP, CAMP - 44,
                    new float[]{0f, 0.45f, 1f},
                    new Color[]{new Color(145, 225, 255), new Color(119, 151, 255),
                            new Color(170, 74, 235)}));
            g.fillPolygon(new int[]{CAMP - 18, CAMP - 8, CAMP + 2, CAMP + 15, CAMP + 20},
                    new int[]{CAMP + 5, CAMP - 26, CAMP - 40 + flicker, CAMP - 18, CAMP + 5}, 5);
            g.setPaint(new GradientPaint(CAMP, CAMP + 7, new Color(216, 249, 255),
                    CAMP, CAMP - 17, new Color(131, 201, 255)));
            g.fillOval(CAMP - 8, CAMP - 17, 17, 24);
        }
        for (Bunny bunny : bunnies) drawBunny(g, bunny);
        for (Point.Double enemy : enemies) {
            int ex = (int)enemy.x, ey = (int)enemy.y;
            g.setColor(new Color(38, 29, 45)); g.fillOval(ex - 16, ey - 18, 32, 36);
            g.setColor(new Color(255, 91, 95)); g.fillOval(ex - 9, ey - 6, 5, 5); g.fillOval(ex + 4, ey - 6, 5, 5);
        }
        if (attackCooldown <= 0) {
            // The resting blade and upper shaft sit behind his head and coat.
            drawScythe(g);
            drawPlayer(g);
            Graphics2D grip = (Graphics2D) g.create();
            grip.translate(x, y);
            grip.scale(playerFacing, 1);
            grip.translate(13, 1);
            grip.rotate(Math.toRadians(-12));
            grip.scale(-1, 1);
            grip.setColor(new Color(237, 197, 160));
            grip.fillOval(-4, -3, 8, 8);
            grip.dispose();
        } else {
            drawPlayer(g);
            drawScythe(g);
        }
        g.setTransform(screenTransform);
        if (darkness) {
            // Keep the player and fire visible while darkening distant forest.
            java.awt.geom.Area shade = new java.awt.geom.Area(new Rectangle(0, 0, getWidth(), getHeight()));
            shade.subtract(new java.awt.geom.Area(new Ellipse2D.Double(getWidth()/2.0 - 115 * ZOOM, getHeight()/2.0 - 115 * ZOOM, 230 * ZOOM, 230 * ZOOM)));
            if (fuel > 0) shade.subtract(new java.awt.geom.Area(new Ellipse2D.Double((CAMP - cameraX - 155) * ZOOM, (CAMP - cameraY - 155) * ZOOM, 310 * ZOOM, 310 * ZOOM)));
            g.setColor(new Color(4, 8, 22, 190)); g.fill(shade);
        }
        drawHud(g);
        if (started && !ended && snackQuestAccepted && interactingBunny == null
                && !questAnnouncementOpen && !questMenuOpen) drawQuestTracker(g);
        drawBunnyPrompt(g, cameraX, cameraY);
        if (interactingBunny != null && interactingBunny.special) {
            drawBunnyCutscene(g);
        } else if (interactingBunny != null) {
            g.setColor(new Color(7, 15, 19, 225));
            g.fillRect(0, 0, getWidth(), getHeight());
            centered(g, "숲속 토끼\nFOREST BUNNY", getHeight() / 2 - 95, 30, new Color(249, 206, 133));
            Graphics2D portrait = (Graphics2D) g.create();
            portrait.translate(getWidth() / 2.0, getHeight() / 2.0 - 20);
            portrait.scale(2, 2);
            drawBunny(portrait, new Bunny(0, 0, 0));
            portrait.dispose();
            centered(g, bunnyPetted ? "토끼가 기분 좋게 귀를 움직입니다!\nThe bunny wiggles its ears happily!" : "호기심 많은 토끼가 당신을 바라봅니다.\nA curious bunny watches you.",
                    getHeight() / 2 + 30, 18, Color.WHITE);
            centered(g, "E: 쓰다듬기     F / Esc: 나가기\nE: pet bunny     F / Esc: leave", getHeight() / 2 + 80, 18, new Color(249, 206, 133));
            centered(g, "일시 정지\nGame paused", getHeight() / 2 + 115, 13, new Color(190, 207, 195));
        }
        if (questAnnouncementOpen || questMenuOpen) drawQuestOverlay(g);
        if (!started) drawTitleScreen(g);
        else if (ended) overlay(g, "숲에 삼켜졌습니다\nTHE FOREST CLAIMED YOU", night + "일째까지 버텼습니다.\nYou survived until day " + night + ".", "모닥불을 피워 괴물들을 막으세요.\nKeep fueled campfires between you and the creatures.", "R을 눌러 다시 시작\nPress R to play again");
        g.dispose();
    }

    void drawQuestTracker(Graphics2D graphics) {
        Graphics2D g = (Graphics2D) graphics.create();
        int width = 290;
        int left = getWidth() - width - 18;
        int top = 144;
        g.setColor(new Color(8, 13, 24, 205));
        g.fillRoundRect(left, top, width, 150, 14, 14);
        g.setColor(new Color(163, 145, 210, 170));
        g.fillRoundRect(left, top + 15, 3, 120, 3, 3);
        label(g, "현재 퀘스트\nCurrent quest", left + 17, top + 25, 13, new Color(184, 177, 211));
        label(g, "Q: 메뉴\nQ: menu", left + width - 72, top + 25, 12, new Color(184, 177, 211));
        label(g, SNACK_QUEST_TITLE, left + 17, top + 66, 17, new Color(241, 232, 255));
        label(g, "코네히토에게 줄 먹이를 찾으세요.\nFind food for Conejito.",
                left + 17, top + 113, 14, new Color(201, 213, 228));
        g.dispose();
    }

    void drawQuestOverlay(Graphics2D graphics) {
        Graphics2D g = (Graphics2D) graphics.create();
        if (questAnnouncementOpen) {
            double progress = Math.clamp(questFadeTime / 0.6, 0, 1);
            float opacity = (float)(progress * progress * (3 - 2 * progress));
            g.setComposite(AlphaComposite.SrcOver.derive(opacity));
        }
        g.setColor(new Color(0, 0, 0, 215));
        g.fillRect(0, 0, getWidth(), getHeight());
        int middle = getHeight() / 2;
        centered(g, questAnnouncementOpen ? "Quest Accepted:" : "퀘스트\nQuests", middle - 155, 36,
                new Color(231, 221, 250));
        if (snackQuestAccepted) {
            drawQuestTitle(g, SNACK_QUEST_TITLE, middle - 65, questAnnouncementOpen);
            centered(g, "코네히토에게 줄 먹이를 찾으세요.\nFind food for Conejito.", middle + 10, 18,
                    new Color(197, 210, 228));
        } else {
            centered(g, "아직 퀘스트가 없습니다. 숲을 탐험해 보세요.\nNo quests yet. Explore the forest.", middle, 18, new Color(197, 210, 228));
        }
        centered(g, questAnnouncementOpen ? "Enter: 계속     Q: 퀘스트 메뉴\nEnter: continue     Q: quest menu" : "Q / Esc / Enter: 닫기\nQ / Esc / Enter: close",
                middle + 155, 16, new Color(197, 210, 228));
        centered(g, "일시 정지\nGame paused", middle + 210, 12, new Color(150, 162, 185));
        g.dispose();
    }

    // Shared by quest screens: only a new acceptance gets the glow.
    void drawQuestTitle(Graphics2D graphics, String title, int baseline, boolean newlyAccepted) {
        Graphics2D g = (Graphics2D) graphics.create();
        g.setFont(new Font("Dialog", Font.BOLD, 26));
        int left = (getWidth() - g.getFontMetrics().stringWidth(title)) / 2;
        if (newlyAccepted) {
            Shape letters = g.getFont().createGlyphVector(g.getFontRenderContext(), title)
                    .getOutline(left, baseline);
            // Layer translucent outlines to soften the light around the lettering.
            for (int spread = 12; spread >= 2; spread -= 2) {
                g.setStroke(new BasicStroke(spread, BasicStroke.CAP_ROUND, BasicStroke.JOIN_ROUND));
                g.setColor(new Color(158, 172, 255, 12));
                g.draw(letters);
            }
        }
        g.setColor(new Color(244, 239, 255));
        g.drawString(title, left, baseline);
        g.dispose();
    }

    void drawHud(Graphics2D g) {
        g.setColor(new Color(12, 20, 22, 225)); g.fillRoundRect(18, 18, getWidth() - 36, 112, 18, 18);
        label(g, GAME_NAME, 36, 57, 30, new Color(230, 76, 82));
        int seconds = (int)Math.ceil((darkness ? NIGHT_SECONDS : DAY_SECONDS) - phaseTime);
        label(g, (darkness ? "밤 " : "낮 ") + night + "  •  " + (darkness ? "새벽까지 " : "일몰까지 ") + seconds + "초\n"
                + (darkness ? "Night " : "Day ") + night + "  •  " + seconds + "s until " + (darkness ? "dawn" : "sunset"),
                255, 45, 16, new Color(228, 230, 214));
        label(g, "나무: " + wood + "\nWood: " + wood, getWidth() - 140, 45, 16, new Color(228, 230, 214));
        bar(g, 36, 84, health, new Color(145, 225, 255), "체력", "Health");
        bar(g, 310, 84, fuel, new Color(119, 139, 210), "불", "Fire");
        double dist = Math.hypot(x - CAMP, y - CAMP);
        String koDirection = (y > CAMP + 60 ? "북" : y < CAMP - 60 ? "남" : "") + (x > CAMP + 60 ? "서" : x < CAMP - 60 ? "동" : "");
        String direction = (y > CAMP + 60 ? "N" : y < CAMP - 60 ? "S" : "") + (x > CAMP + 60 ? "W" : x < CAMP - 60 ? "E" : "");
        label(g, dist < 105 ? "E: 모닥불에 나무 넣기\nE: add wood to fire" : "야영지: " + (int)dist + "걸음 " + koDirection
                + "\nCamp: " + (int)dist + " steps " + direction, 590, 97, 14, new Color(228, 230, 214));
        g.setColor(new Color(12, 20, 22, 225)); g.fillRoundRect(18, getHeight() - 112, getWidth() - 36, 94, 16, 16);
        label(g, "WASD / 방향키: 이동    E: 채집 / 연료    F: 대화    SPACE: 낫 휘두르기    Q: 퀘스트\n"
                + "WASD / Arrows: move    E: gather / fuel    F: talk    SPACE: swing scythe    Q: quests",
                36, getHeight() - 88, 14, new Color(219, 224, 209));
        if (messageTime > 0) label(g, message, 36, getHeight() - 49, 14, new Color(247, 199, 115));
    }

    void bar(Graphics2D g, int left, int top, double value, Color color, String korean, String english) {
        g.setColor(new Color(57, 65, 61)); g.fillRoundRect(left, top, 160, 17, 8, 8);
        g.setColor(color); g.fillRoundRect(left, top, (int)(160 * value / 100), 17, 8, 8);
        label(g, korean + " " + (int)value + "\n" + english, left + 169, top + 10, 13, new Color(225, 226, 212));
    }

    void label(Graphics2D g, String text, int x, int y, int size, Color color) {
        String[] lines = text.split("\n", 2);
        g.setFont(new Font("Dialog", Font.BOLD, size)); g.setColor(color);
        g.drawString(lines[0], x, y);
        if (lines.length > 1) {
            g.setFont(new Font("Dialog", Font.PLAIN, Math.max(10, size - 5)));
            g.setColor(new Color(color.getRed(), color.getGreen(), color.getBlue(), 190));
            g.drawString(lines[1], x, y + Math.max(14, size - 1));
        }
    }

    void drawTitleScreen(Graphics2D g) {
        g.setColor(new Color(7, 15, 19, 225));
        g.fillRect(0, 0, getWidth(), getHeight());
        int middle = getHeight() / 2;
        centered(g, GAME_NAME, middle - 80, 64, new Color(249, 206, 133));
        Color hintColor = new Color(190, 207, 195);
        centered(g, "그녀를 찾아라.", middle + 5, 28, hintColor);
        centered(g, "find Her.", middle + 45, 28, hintColor);
        Color promptColor = new Color(171, 184, 179);
        centered(g, "Enter를 눌러 시작", getHeight() - 80, 14, promptColor);
        centered(g, "Press ENTER to start", getHeight() - 60, 12, promptColor);
    }

    void overlay(Graphics2D g, String title, String subtitle, String hint, String action) {
        g.setColor(new Color(7, 15, 19, 225)); g.fillRect(0, 0, getWidth(), getHeight());
        int middle = getHeight() / 2;
        centered(g, title, middle - 150, 36, new Color(249, 206, 133));
        centered(g, subtitle, middle - 55, 22, Color.WHITE);
        centered(g, hint, middle + 20, 16, new Color(190, 207, 195));
        if (started) centered(g, "WASD / 방향키: 이동  •  E: 상호작용  •  SPACE: 공격\nWASD / Arrows to move  •  E to interact  •  SPACE to attack", middle + 85, 15, new Color(190, 207, 195));
        centered(g, action, middle + 170, 21, new Color(249, 206, 133));
    }

    void centered(Graphics2D g, String text, int y, int size, Color color) {
        String[] lines = text.split("\n", 2);
        g.setFont(new Font("Dialog", Font.BOLD, size)); g.setColor(color);
        g.drawString(lines[0], (getWidth() - g.getFontMetrics().stringWidth(lines[0]))/2, y);
        if (lines.length > 1) {
            g.setFont(new Font("Dialog", Font.PLAIN, Math.max(10, size - 6)));
            g.setColor(new Color(color.getRed(), color.getGreen(), color.getBlue(), 190));
            g.drawString(lines[1], (getWidth() - g.getFontMetrics().stringWidth(lines[1]))/2,
                    y + Math.max(14, size - 2));
        }
    }
}
