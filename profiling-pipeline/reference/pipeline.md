perf stat -d ./flame_vj -r runcard_WmJ_26feb26.run --all-in-one

perf record -F 999 -g --call-graph dwarf ./flame_vj -r runcard_WmJ_26feb26.run --all-in-one

perf annotate -s 'FLAME::MatrixElementVJ<double>::get_squared_me'

perf report --children

perf script > out.perf 

stackcollapse-perf.pl out.perf > out.folded

flamegraph.pl out.folded > flame.svg

firefox flame.svg 

heaptrack ./flame_vj -r runcard_WmJ_26feb26.run --all-in-one

gprof2dot -f perf out.perf -s -z 'FLAME::CrossSection::dsigma*'  --depth 10 -n 1 -e 0.5 -o callgraph.dot

dot -Grankdir=TB -Tpdf callgraph.dot -o callgraph.pdf
