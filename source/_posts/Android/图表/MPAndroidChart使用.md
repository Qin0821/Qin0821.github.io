### 通用设置

开关图表描述
Chart.getDescription().setEnabled(bool)

开关图表图例
Chart.getLegend().setEnabled(bool)

开关右侧Y轴
Chart.getAxisRight().setEnabled(bool)

Y轴最小值(貌似没有效果)
Chart.getAxisLeft().setAxisMinimum(float)

Y轴最小值（已废弃，但是有效果）
Chart.getAxisLeft().setAxisMinValue(float)

X轴最小步长
Chart.getXAxis().setGranularity(float)

格式化X轴刻度
ValueFormatter formatter = new ValueFormatter() {
            @Override
            public String getAxisLabel(float value, AxisBase axis) {
                return (int) value + "厘米";
            }
        };
Chart.getXAxis().setValueFormatter(formatter)







### 折线图

开关每个点数值显示
LineDataSet.setDrawValues(bool)

每个数据点是否画圈
LineDataSet.setDrawCircles(bool)

设置折线模式
LineDataSet.setMode(LineDateSet.Mode)

Mode:
    LINEAR
    STEPPED
    CUBIC_BEZIER 数据点附近曲线处理，数据点不一定是最大（最小）值，曲线更加顺滑
    HORIZONTAL_BEZIER 数据点附近曲线处理，数据点一定是最大（最小）值，数据更加准确，有时看起来有点别扭
