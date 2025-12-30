# Lookup tables

In normal programs, you can trade memory for CPU to improve performance, by pre-computing
and storing lookup tables for some part of the computation. We can do the same thing in
halo2 circuits!

A lookup table can be thought of as enforcing a *relation* between variables, where the relation is expressed as a table.
Assuming we have only one lookup argument in our constraint system, the total size of tables is constrained by the size of the circuit:
each table entry costs one row, and it also costs one row to do each lookup.

TODO

# 查找表

在普通程序中，您可以通过预先计算并存储查找表来以内存换取 CPU 性能，从而优化计算。在 halo2 电路中，我们也可以做同样的事情！

查找表可以被视为在变量之间强制执行一种*关系*，这种关系通过表格来表达。假设我们的约束系统中只有一个查找参数，表格的总大小受电路大小的限制：每个表格条目占用一行，每次查找也占用一行。