cat > frontend/src/components/RciDashboard.tsx <<'EOF'
import React, { useEffect, useState } from "react";
import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
  ReferenceLine,
  Label,
} from "recharts";

function CustomTooltip({ active, payload, label }: any) {
  if (active && payload && payload.length) {
    const { rci, notes } = payload[0].payload;
    return (
      <div className="bg-white p-3 border rounded shadow text-sm">
        <p className="font-semibold">{label}</p>
        <p>
          RCI Score: <span className="font-bold">{rci}</span>
        </p>
        {notes && <p className="text-gray-600 italic">"{notes}"</p>}
      </div>
    );
  }
  return null;
}

export default function RciDashboard({ residentId }: { residentId: string }) {
  const [data, setData] = useState<any[]>([]);
  const [avgRci, setAvgRci] = useState<number | null>(null);
  const [latestRci, setLatestRci] = useState<number | null>(null);
  const [trend, setTrend] = useState<string>("");

  useEffect(() => {
    fetch(`/api/residents/${residentId}/assessments`)
      .then((res) => res.json())
      .then((d) => {
        const formatted = d.map((a: any, i: number) => ({
          name: `Assessment ${i + 1}`,
          rci: a.rci_score,
          notes: a.notes,
        }));
        setData(formatted);

        if (formatted.length > 0) {
          const avg =
            formatted.reduce((sum, x) => sum + Number(x.rci), 0) /
            formatted.length;
          setAvgRci(Number(avg.toFixed(1)));

          const last = formatted[formatted.length - 1].rci;
          setLatestRci(last);

          if (formatted.length > 1) {
            const prev = formatted[formatted.length - 2].rci;
            if (last > prev) setTrend("improving");
            else if (last < prev) setTrend("declining");
            else setTrend("stable");
          }
        }
      });
  }, [residentId]);

  // Split dataset into ranges
  const redData = data.map((d) => (d.rci < 60 ? d : { ...d, rci: null }));
  const yellowData = data.map((d) =>
    d.rci >= 60 && d.rci < 75 ? d : { ...d, rci: null }
  );
  const greenData = data.map((d) => (d.rci >= 75 ? d : { ...d, rci: null }));

  return (
    <div className="p-4 border rounded shadow bg-white mt-6">
      <h2 className="text-xl font-bold mb-4">RCI Progress</h2>

      {/* Summary Card */}
      {latestRci !== null && avgRci !== null && (
        <div className="grid grid-cols-3 gap-4 mb-6 text-center">
          <div className="p-4 border rounded shadow bg-gray-50">
            <p className="text-sm text-gray-600">Latest RCI</p>
            <p className="text-xl font-bold">{latestRci}</p>
          </div>
          <div className="p-4 border rounded shadow bg-gray-50">
            <p className="text-sm text-gray-600">Average RCI</p>
            <p className="text-xl font-bold">{avgRci}</p>
          </div>
          <div className="p-4 border rounded shadow bg-gray-50">
            <p className="text-sm text-gray-600">Trend</p>
            <p className="text-xl font-bold">
              {trend === "improving" && "📈 Improving"}
              {trend === "declining" && "📉 Declining"}
              {trend === "stable" && "➡️ Stable"}
            </p>
          </div>
        </div>
      )}

      {/* Chart */}
      {data.length > 0 ? (
        <ResponsiveContainer width="100%" height={300}>
          <LineChart data={data}>
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis dataKey="name" />
            <YAxis domain={[0, 100]} />
            <Tooltip content={<CustomTooltip />} />

            {/* Threshold markers */}
            <ReferenceLine y={60} stroke="red" strokeDasharray="3 3">
              <Label value="Low Risk Threshold (60)" position="insideTopLeft" fill="red" fontSize={12} />
            </ReferenceLine>
            <ReferenceLine y={75} stroke="orange" strokeDasharray="3 3">
              <Label value="High Recovery Threshold (75)" position="insideTopLeft" fill="orange" fontSize={12} />
            </ReferenceLine>

            {/* Average RCI line */}
            {avgRci !== null && (
              <ReferenceLine y={avgRci} stroke="#3b82f6" strokeDasharray="5 5">
                <Label value={`Average RCI (${avgRci})`} position="insideTopLeft" fill="#3b82f6" fontSize={12} />
              </ReferenceLine>
            )}

            {/* Dynamic color segments */}
            <Line
              type="monotone"
              dataKey="rci"
              data={redData}
              stroke="#dc2626"
              strokeWidth={2}
              dot={{ stroke: "#dc2626", r: 4 }}
              connectNulls
            />
            <Line
              type="monotone"
              dataKey="rci"
              data={yellowData}
              stroke="#facc15"
              strokeWidth={2}
              dot={{ stroke: "#facc15", r: 4 }}
              connectNulls
            />
            <Line
              type="monotone"
              dataKey="rci"
              data={greenData}
              stroke="#16a34a"
              strokeWidth={2}
              dot={{ stroke: "#16a34a", r: 4 }}
              connectNulls
            />
          </LineChart>
        </ResponsiveContainer>
      ) : (
        <p className="text-gray-500">No assessments yet.</p>
      )}
    </div>
  );
}
EOF
