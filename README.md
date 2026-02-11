import javax.swing.*;
import java.awt.*;
import java.awt.event.*;
import java.util.*;

public class Main {

    static class Cell {
        boolean isMine = false;
        boolean isRevealed = false;
        boolean isFlagged = false;
        int adjacentMines = 0;
    }

    static class Board {
        int rows, cols, totalMines, revealedCells;
        Cell[][] grid;
        boolean gameOver = false;
        boolean win = false;
        Random rng = new Random();

        Board(int r, int c, int m) {
            rows = r; cols = c; totalMines = m;
            revealedCells = 0;
            grid = new Cell[rows][cols];
            for (int i = 0; i < rows; i++)
                for (int j = 0; j < cols; j++)
                    grid[i][j] = new Cell();
        }

        boolean inBounds(int r, int c) {
            return r >= 0 && r < rows && c >= 0 && c < cols;
        }

        void placeMines(int firstRow, int firstCol) {
            int placed = 0;
            while (placed < totalMines) {
                int r = rng.nextInt(rows);
                int c = rng.nextInt(cols);
                if ((r == firstRow && c == firstCol) || grid[r][c].isMine) continue;
                grid[r][c].isMine = true;
                placed++;
            }
        }

        void calculateAdjacents() {
            int[] dr = {-1,-1,-1,0,0,1,1,1};
            int[] dc = {-1,0,1,-1,1,-1,0,1};

            for (int r = 0; r < rows; r++) {
                for (int c = 0; c < cols; c++) {
                    if (grid[r][c].isMine) continue;
                    int count = 0;
                    for (int d = 0; d < 8; d++) {
                        int nr = r + dr[d], nc = c + dc[d];
                        if (inBounds(nr, nc) && grid[nr][nc].isMine) count++;
                    }
                    grid[r][c].adjacentMines = count;
                }
            }
        }

        void revealBFS(int row, int col) {
            int[] dr = {-1,-1,-1,0,0,1,1,1};
            int[] dc = {-1,0,1,-1,1,-1,0,1};
            ArrayDeque<int[]> q = new ArrayDeque<>();
            q.add(new int[]{row, col});

            while (!q.isEmpty()) {
                int[] cur = q.poll();
                int r = cur[0], c = cur[1];

                if (!inBounds(r, c)) continue;
                if (grid[r][c].isRevealed || grid[r][c].isFlagged) continue;

                grid[r][c].isRevealed = true;
                revealedCells++;

                if (grid[r][c].adjacentMines == 0) {
                    for (int d = 0; d < 8; d++) {
                        int nr = r + dr[d], nc = c + dc[d];
                        if (inBounds(nr, nc) && !grid[nr][nc].isRevealed) {
                            q.add(new int[]{nr, nc});
                        }
                    }
                }
            }
        }

        void revealAllMines() {
            for (int r = 0; r < rows; r++)
                for (int c = 0; c < cols; c++)
                    if (grid[r][c].isMine)
                        grid[r][c].isRevealed = true;
        }
    }

    // ======= SWING FRONTEND =======
    static class MinesweeperUI extends JFrame {
        int rows, cols, mines;
        Board board;
        JButton[][] buttons;
        boolean firstMove = true;

        JLabel status = new JLabel("Left click: Reveal | Right click: Flag");

        MinesweeperUI(int r, int c, int m) {
            rows = r; cols = c; mines = m;
            setTitle("MiniSwiper - Minesweeper");
            setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
            setLayout(new BorderLayout());
            setResizable(false);

            JButton restartBtn = new JButton("Restart");
            restartBtn.addActionListener(e -> resetGame());

            JPanel top = new JPanel(new BorderLayout());
            top.setBorder(BorderFactory.createEmptyBorder(8, 10, 8, 10));
            top.add(status, BorderLayout.WEST);
            top.add(restartBtn, BorderLayout.EAST);

            add(top, BorderLayout.NORTH);

            JPanel gridPanel = new JPanel();
            gridPanel.setLayout(new GridLayout(rows, cols, 2, 2));
            gridPanel.setBorder(BorderFactory.createEmptyBorder(10, 10, 10, 10));

            buttons = new JButton[rows][cols];

            for (int i = 0; i < rows; i++) {
                for (int j = 0; j < cols; j++) {
                    JButton btn = new JButton("");
                    btn.setPreferredSize(new Dimension(45, 45));
                    btn.setFont(new Font("Arial", Font.BOLD, 16));
                    btn.setFocusPainted(false);
                    btn.setBackground(new Color(40, 40, 40));
                    btn.setForeground(Color.WHITE);

                    final int rr = i, cc = j;

                    btn.addMouseListener(new MouseAdapter() {
                        @Override
                        public void mousePressed(MouseEvent e) {
                            if (board.gameOver) return;

                            // RIGHT CLICK -> FLAG
                            if (SwingUtilities.isRightMouseButton(e)) {
                                toggleFlag(rr, cc);
                            }
                            // LEFT CLICK -> REVEAL
                            else if (SwingUtilities.isLeftMouseButton(e)) {
                                reveal(rr, cc);
                            }
                            refreshUI();
                            checkEnd();
                        }
                    });

                    buttons[i][j] = btn;
                    gridPanel.add(btn);
                }
            }

            add(gridPanel, BorderLayout.CENTER);

            pack();
            setLocationRelativeTo(null);
            resetGame();
            setVisible(true);
        }

        void resetGame() {
            board = new Board(rows, cols, mines);
            firstMove = true;
            status.setText("Left click: Reveal | Right click: Flag");
            for (int r = 0; r < rows; r++) {
                for (int c = 0; c < cols; c++) {
                    buttons[r][c].setEnabled(true);
                    buttons[r][c].setText("");
                    buttons[r][c].setBackground(new Color(40, 40, 40));
                }
            }
        }

        void toggleFlag(int r, int c) {
            Cell cell = board.grid[r][c];
            if (cell.isRevealed) return;
            cell.isFlagged = !cell.isFlagged;
        }

        void reveal(int r, int c) {
            Cell cell = board.grid[r][c];
            if (cell.isFlagged || cell.isRevealed) return;

            if (firstMove) {
                board.placeMines(r, c);
                board.calculateAdjacents();
                firstMove = false;
            }

            if (cell.isMine) {
                board.gameOver = true;
                board.revealAllMines();
                status.setText("💥 You hit a mine!");
                return;
            }

            board.revealBFS(r, c);

            if (board.revealedCells == rows * cols - mines) {
                board.win = true;
                board.gameOver = true;
                board.revealAllMines();
                status.setText("🎉 You Win!");
            }
        }

        void refreshUI() {
            for (int r = 0; r < rows; r++) {
                for (int c = 0; c < cols; c++) {
                    Cell cell = board.grid[r][c];
                    JButton btn = buttons[r][c];

                    if (cell.isRevealed) {
                        btn.setEnabled(false);
                        btn.setBackground(new Color(70, 70, 70));
                        if (cell.isMine) {
                            btn.setText("💣");
                        } else if (cell.adjacentMines > 0) {
                            btn.setText(String.valueOf(cell.adjacentMines));
                        } else {
                            btn.setText("");
                        }
                    } else {
                        btn.setEnabled(true);
                        btn.setBackground(new Color(40, 40, 40));
                        btn.setText(cell.isFlagged ? "🚩" : "");
                    }
                }
            }
        }

        void checkEnd() {
            if (!board.gameOver) return;

            refreshUI();
            if (board.win) {
                JOptionPane.showMessageDialog(this, "Congratulations! You Win! 🎉");
            } else {
                JOptionPane.showMessageDialog(this, "Game Over! You hit a mine 💥");
            }
        }
    }

    public static void main(String[] args) {
        // You can change these
        int rows = 9, cols = 9, mines = 10;

        SwingUtilities.invokeLater(() -> new MinesweeperUI(rows, cols, mines));
    }
}

