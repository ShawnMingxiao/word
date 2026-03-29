import java.time.LocalDate;
import java.time.YearMonth;
import java.time.format.DateTimeFormatter;
import java.util.*;

public class DateFixer {

    public static void main(String[] args) {
        // 这里可以随意替换成你真实的日期列表（任意长度、任意年份、任意缺失/重复情况）
        List<String> dates = Arrays.asList(
            "20251130", "20251230", "20260131",
            "20260301", "20260331",
            "20260428", "20260530",
            "20260702", "20260731"
        );

        System.out.println("原始数据: " + dates);
        List<String> result = fixMissingMonths(dates);
        System.out.println("处理后结果: " + result);
    }

    /**
     * 核心方法：完全通用
     * - 自动检测任意月份缺失（gap）
     * - 如果下一个月有多条数据，则把下个月的【第一条】（按输入顺序出现的第一条）修改成缺失月份的最后一天
     * - 支持连续多个月份缺失（会依次借用，只要下个月有多余数据）
     * - 最终确保每个月只有一条数据
     * - 支持任意年份、任意输入顺序、无效日期自动跳过
     */
    public static List<String> fixMissingMonths(List<String> inputDates) {
        if (inputDates == null || inputDates.isEmpty()) {
            return new ArrayList<>();
        }

        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyyMMdd");

        // 按输入顺序分组（同一个月内的日期按第一次出现顺序保存，便于“第一条”）
        Map<YearMonth, List<LocalDate>> groups = new LinkedHashMap<>();

        for (String str : inputDates) {
            if (str == null || str.length() != 8 || !str.matches("\\d{8}")) continue;

            try {
                LocalDate date = LocalDate.parse(str, formatter);
                // 通用修正：如果输入日期天数超过当月最大天数，先修正为月末
                YearMonth ym = YearMonth.from(date);
                if (date.getDayOfMonth() > ym.lengthOfMonth()) {
                    date = ym.atEndOfMonth();
                }
                groups.computeIfAbsent(ym, k -> new ArrayList<>()).add(date);
            } catch (Exception e) {
                System.err.println("跳过无效日期: " + str);
            }
        }

        if (groups.isEmpty()) return new ArrayList<>();

        // 获取所有月份并按时间顺序排序（用于检测缺失）
        List<YearMonth> sortedMonths = new ArrayList<>(groups.keySet());
        Collections.sort(sortedMonths);

        // 最终结果（按处理顺序插入，保证输出是时间顺序）
        Map<YearMonth, String> resultMap = new LinkedHashMap<>();

        YearMonth previous = null;

        for (YearMonth current : sortedMonths) {
            // 检测并处理缺失的月份
            if (previous != null) {
                YearMonth expected = previous.plusMonths(1);
                while (expected.isBefore(current)) {
                    List<LocalDate> currentList = groups.get(current);
                    // 关键条件：只有当下个月有多条数据（size > 1）时才借用，保证当前月至少留一条
                    if (currentList != null && currentList.size() > 1) {
                        LocalDate borrowedDate = currentList.remove(0); // 借用输入顺序的第一条
                        LocalDate fixedDate = expected.atEndOfMonth(); // 改为缺失月份的最后一天
                        resultMap.put(expected, fixedDate.format(formatter));
                    }
                    expected = expected.plusMonths(1);
                }
            }

            // 处理当前月份（借用后剩下的第一条）
            List<LocalDate> currentList = groups.get(current);
            if (currentList != null && !currentList.isEmpty()) {
                LocalDate keptDate = currentList.get(0);
                resultMap.put(current, keptDate.format(formatter));
            }

            previous = current;
        }

        return new ArrayList<>(resultMap.values());
    }
}
