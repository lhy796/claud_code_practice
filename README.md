import java.nio.file.*;
import java.util.*;

/** 간단한 BFS 버전. 통신과 파싱은 GameSupport.java에 있다. */
public class Main {
    static final int[] DR = {-1, 1, 0, 0}, DC = {0, 0, -1, 1};
    static final String[] DIR = {"U", "D", "L", "R"};

    public static void main(String[] args) throws Exception {
        Bot bot = new Bot(-1);
        if (args.length == 2 && args[0].equals("--offline")) {
            System.out.println(bot.decide(Board.parse(Files.readString(Path.of(args[1])))));
            return;
        }
        if (args.length != 0) throw new IllegalArgumentException("사용법: Main [--offline FILE]");
        try (GameSupport game = new GameSupport()) {
            Board board;
            while ((board = game.receive()) != null) {
                String command = bot.decide(board);
                System.out.println(command);
                game.send(command); // 이 명령을 받으면 실제 게임의 탱크가 행동한다.
            }
        }
    }

    static class Bot {
        Board b;
        boolean hasSupply;
        final Cipher cipher;
        Bot(int shift) { cipher = new Cipher(shift); }

        // 1. 쏠 수 있으면 쏜다. 2. 탄약이 부족하면 보급. 3. BFS로 길을 찾는다.
        String decide(Board board) {
            b = board;
            cipher.observe(b);
            hasSupply = Arrays.asList(b.cells).contains("F");
            if (b.target < 0 || b.hp <= 0) return "S";
            int direction = aim(b.start, false);
            if (direction >= 0 && damage() > 0 && (damage() >= b.targetHp() || hasSupply)) return fire(direction);

            boolean supplyNeeded = damage() < b.targetHp();
            // 보급소에 이미 붙어 있다면 장애물을 위한 여유 메가탄 1발도 확보한다.
            if (b.adjacentSupply(b.start) && damage() < b.targetHp() + 70) supplyNeeded = true;
            if (supplyNeeded) {
                if (b.adjacentSupply(b.start) && !b.codes.isEmpty()) return "G " + cipher.decode(b);
                String action = bfs(true, false); // 먼저 장애물 없는 보급 경로
                if (action == null) action = bfs(true, true);
                if (action != null) return action;
            }
            if (damage() < b.targetHp()) return "S";
            String action = bfs(false, false); // 먼저 나무·적을 피해 가는 길
            if (action == null) action = bfs(false, true); // 없으면 부수며 가는 길
            return action == null ? "S" : action;
        }

        // 큐에 [현재 칸, 출발점에서 처음 갈 방향, 남은 체력]을 넣는다.
        String bfs(boolean toSupply, boolean breakObstacles) {
            Queue<int[]> queue = new ArrayDeque<>();
            int[] bestHp = new int[b.cells.length];
            Arrays.fill(bestHp, -1);
            queue.add(new int[]{b.start, -1, b.hp});
            bestHp[b.start] = b.hp;
            while (!queue.isEmpty()) {
                int[] current = queue.poll();
                int p = current[0], first = current[1], hp = current[2];
                int aimDirection = toSupply ? -1 : aim(p, breakObstacles);
                boolean arrived = toSupply ? b.adjacentSupply(p) : aimDirection >= 0;
                if (arrived) {
                    if (first == -1) return toSupply ? null : fire(aimDirection);
                    int next = b.step(b.start, first);
                    return destructible(b.cells[next]) ? fire(first) : DIR[first] + " A";
                }
                for (int d = 0; d < 4; d++) {
                    int q = b.step(p, d);
                    if (q < 0) continue;
                    String tile = b.cells[q];
                    boolean ground = tile.equals("G") || tile.equals("S") || tile.equals("M");
                    boolean canBreak = breakObstacles && canBreak(tile);
                    if (!ground && !canBreak) continue;
                    int nextHp = hp - (tile.equals("S") ? 10 : 0);
                    if (nextHp <= 0 || nextHp <= bestHp[q]) continue;
                    bestHp[q] = nextHp;
                    queue.add(new int[]{q, first == -1 ? d : first, nextHp});
                }
            }
            return null;
        }

        // 이 칸에서 적 포탑 X가 사거리 3칸 안에 있는지 확인한다.
        int aim(int position, boolean breakObstacles) {
            for (int d = 0; d < 4; d++) {
                int p = position;
                for (int distance = 1; distance <= 3; distance++) {
                    p = b.step(p, d);
                    if (p < 0) break;
                    String tile = b.cells[p];
                    if (tile.equals("X")) return d;
                    if (tile.equals("G") || tile.equals("S") || tile.equals("W") || tile.equals("M")) continue;
                    if (breakObstacles && canBreak(tile)) continue;
                    break; // 바위·보급소·아군·제거할 수 없는 장애물에 차단
                }
            }
            return -1;
        }

        String fire(int direction) {
            int p = b.start;
            for (int distance = 1; distance <= 3; distance++) {
                p = b.step(p, direction);
                if (p < 0) return "S";
                String tile = b.cells[p];
                if (tile.equals("G") || tile.equals("S") || tile.equals("W")) continue;
                if (!destructible(tile) && !tile.equals("X")) return "S";
                if (b.mega > 0 && (b.normal == 0 || objectHp(tile) > 30)) return DIR[direction] + " F M";
                return b.normal > 0 ? DIR[direction] + " F" : "S";
            }
            return "S";
        }
        boolean canBreak(String tile) {
            return destructible(tile) && damage() >= objectHp(tile) + (hasSupply ? 0 : b.targetHp());
        }
        int damage() { return b.normal * 30 + b.mega * 70; }
        boolean destructible(String tile) { return tile.equals("T") || tile.matches("E[1-3]"); }
        int objectHp(String tile) { return tile.equals("T") ? 30 : b.health.getOrDefault(tile, 100); }
    }
}
