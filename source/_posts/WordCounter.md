---
title: WordCounter
date: 2022-03-18 10:45:31
tags: Design
---

```
import java.util.Spliterator;
import java.util.function.Consumer;
import java.util.stream.IntStream;
import java.util.stream.Stream;
import java.util.stream.StreamSupport;

/**
 * 计算单词个数
 * <U> U reduce(U identity,
 * BiFunction<U, ? super T, U> accumulator,
 * BinaryOperator<U> combiner);
 */
public class WordCounter {
    private final boolean lastSpace;
    private final int count;

    public WordCounter(boolean lastSpace, int count) {
        this.lastSpace = lastSpace;
        this.count = count;
    }

    /**
     * 和迭代算法一样 ，
     * accumulate 方法一
     * 个个遍历Character
     *
     * @param character
     * @return
     */
    public WordCounter accumulator(Character character) {
        if (Character.isWhitespace(character)) {
            return lastSpace ? this : new WordCounter(true, count);
        } else {
            return lastSpace ? new WordCounter(false, count + 1) : this;
        }
    }

    /**
     * 合并两个Word Counter，把其
     * 计数器加起来
     *
     * @param wordCounter
     * @return
     */
    public WordCounter combiner(WordCounter wordCounter) {
        return new WordCounter(lastSpace, count + wordCounter.count);
    }

    public int getCount() {
        return count;
    }

    public static void main(String[] args) {
        String str = " Nel mezzo del cammin di nostra vita " +
                "mi ritrovai in una selva oscura" +
                " ché la dritta via era smarrita ";
        Stream<Character> characterStream = IntStream.range(0, str.length()).mapToObj(str::charAt);
        System.out.println(countWordsIteratively(characterStream));
        Spliterator<Character> spliterator = new WordCounterSpliterator(str);
        Stream<Character> characterSpliteratorStream = StreamSupport.stream(spliterator, true);
        System.out.println(countWordsIteratively(characterSpliteratorStream.parallel()));
        System.out.println(countWordsIteratively(str));


    }

    public static int countWordsIteratively(Stream<Character> characterStream) {
        WordCounter wordCounter =
                characterStream.reduce(
                        new WordCounter(true, 0), WordCounter::accumulator, WordCounter::combiner);
        return wordCounter.getCount();
    }

    public static int countWordsIteratively(String str) {
        boolean lastSpace = true;
        int count = 0;
        char[] chars = str.toCharArray();
        for (int i = 0; i < chars.length; i++) {
            if (Character.isWhitespace(chars[i])) {
                lastSpace = true;
            } else {
                if (lastSpace) count++;
                lastSpace = false;
            }
        }
        return count;
    }

    public static class WordCounterSpliterator implements Spliterator<Character> {

        private final String string;
        private int currentChar = 0;

        public WordCounterSpliterator(String string) {
            this.string = string;
        }

        /**
         * tryAdvance方法的行为类似于普通的
         * Iterator，因为它会按顺序一个一个使用Spliterator中的元素，并且如果还有其他元素要遍
         * 历就返回true。
         *
         * @param action
         * @return
         */
        @Override
        public boolean tryAdvance(Consumer<? super Character> action) {
            action.accept(string.charAt(currentChar++));
            return currentChar < string.length();
        }

        /**
         * 但trySplit是专为Spliterator接口设计的，因为它可以把一些元素划出去分
         * 给第二个Spliterator（由该方法返回），让它们两个并行处理。
         * 如果你需要执行拆分，就把试探的拆分位置设在要解析的String块的中间。但我
         * 们没有直接使用这个拆分位置，因为要避免把词在中间断开，于是就往前找，直到找到
         * 一个空格。一旦找到了适当的拆分位置，就可以创建一个新的Spliterator来遍历从
         * 当前位置到拆分位置的子串；
         *
         * @return
         */
        @Override
        public Spliterator<Character> trySplit() {
            int currentSize = string.length() - currentChar;
            if (currentSize < 10) {
                return null;
            }
            for (int splitPos = currentSize / 2 + currentChar; splitPos < string.length(); splitPos++) {
                if (Character.isWhitespace(string.charAt(splitPos))) {
                    Spliterator<Character> spliterator =
                            new WordCounterSpliterator(string.substring(currentChar, splitPos));
                    currentChar = splitPos;
                    return spliterator;
                }
            }
            return null;
        }

        /**
         * Spliterator还可通过
         * estimateSize方法估计还剩下多少元素要遍历，因为即使不那么确切，能快速算出来是一个值
         * 也有助于让拆分均匀一点。
         * 还需要遍历的元素的estimatedSize就是这个Spliterator解析的String的总长度和
         * 当前遍历的位置的差。
         *
         * @return
         */
        @Override
        public long estimateSize() {
            return string.length() - currentChar;
        }

        /**
         * Spliterator是ORDERED（顺序就是String
         * 中各个Character的次序）、SIZED（estimatedSize方法的返回值是精确的）、
         * SUBSIZED（trySplit方法创建的其他Spliterator也有确切大小）、NONNULL（String
         * 中不能有为 null 的 Character ） 和 IMMUTABLE （在解析 String 时不能再添加
         * Character，因为String本身是一个不可变类）的
         *
         * @return
         */
        @Override
        public int characteristics() {
            return ORDERED + SIZED + SUBSIZED + NONNULL + IMMUTABLE;
        }
    }
}
```

