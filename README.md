import { useEffect, useMemo, useRef, useState } from 'react';

type SceneMode = '2D' | '2.5D' | '3D';
type LayerType = 'sprite' | 'box' | 'plane' | 'light';

type Keyframe = {
  x: number;
  y: number;
  z: number;
  rotation: number;
  scale: number;
};

type Layer = {
  id: string;
  name: string;
  type: LayerType;
  color: string;
  x: number;
  y: number;
  z: number;
  rotation: number;
  scale: number;
  depth: number;
  keyframes: Record<number, Keyframe>;
};

const totalFrames = 24;

const makeLayer = (
  name: string,
  type: LayerType,
  color: string,
  defaults: Partial<Keyframe> & { depth?: number },
): Layer => ({
  id: `${name.toLowerCase().replace(/\s+/g, '-')}-${Math.random().toString(36).slice(2, 8)}`,
  name,
  type,
  color,
  x: defaults.x ?? 0,
  y: defaults.y ?? 0,
  z: defaults.z ?? 0,
  rotation: defaults.rotation ?? 0,
  scale: defaults.scale ?? 1,
  depth: defaults.depth ?? 1,
  keyframes: {},
});

const initialLayers: Layer[] = [
  makeLayer('Hero', 'sprite', '#7dd3fc', { x: -180, y: 30, z: 0, rotation: 0, scale: 1.2, depth: 1 }),
  makeLayer('Ground', 'plane', '#34d399', { x: 0, y: 170, z: 0, rotation: 0, scale: 1.5, depth: 0.75 }),
  makeLayer('Cube', 'box', '#fbbf24', { x: 180, y: -30, z: 40, rotation: 30, scale: 1.1, depth: 1.3 }),
];

const toKeyframe = (layer: Layer): Keyframe => ({
  x: layer.x,
  y: layer.y,
  z: layer.z,
  rotation: layer.rotation,
  scale: layer.scale,
});

const getFramePosition = (layer: Layer, frame: number): Keyframe => {
  const keyframes = Object.entries(layer.keyframes).sort(([a], [b]) => Number(a) - Number(b));
  const currentFrame = Number(frame);

  if (keyframes.length === 0) {
    return toKeyframe(layer);
  }

  const exact = keyframes.find(([f]) => Number(f) === currentFrame);
  if (exact) {
    return {
      x: exact[1].x,
      y: exact[1].y,
      z: exact[1].z,
      rotation: exact[1].rotation,
      scale: exact[1].scale,
    };
  }

  let previous = keyframes[0];
  let next = keyframes[keyframes.length - 1];

  for (let i = 0; i < keyframes.length - 1; i += 1) {
    const [frameA, valueA] = keyframes[i];
    const [frameB, valueB] = keyframes[i + 1];
    const fA = Number(frameA);
    const fB = Number(frameB);

    if (currentFrame >= fA && currentFrame <= fB) {
      previous = [frameA, valueA];
      next = [frameB, valueB];
      break;
    }
  }

  const range = Number(next[0]) - Number(previous[0]);
  const ratio = range === 0 ? 0 : (currentFrame - Number(previous[0])) / range;

  const interpolate = (a: number, b: number) => a + (b - a) * ratio;

  return {
    x: interpolate(previous[1].x, next[1].x),
    y: interpolate(previous[1].y, next[1].y),
    z: interpolate(previous[1].z, next[1].z),
    rotation: interpolate(previous[1].rotation, next[1].rotation),
    scale: interpolate(previous[1].scale, next[1].scale),
  };
};

const clamp = (value: number, min: number, max: number) => Math.min(Math.max(value, min), max);

function App() {
  const canvasRef = useRef<HTMLCanvasElement | null>(null);
  const [sceneMode, setSceneMode] = useState<SceneMode>('2.5D');
  const [layers, setLayers] = useState<Layer[]>(initialLayers);
  const [selectedLayerId, setSelectedLayerId] = useState<string>(initialLayers[0].id);
  const [currentFrame, setCurrentFrame] = useState(0);
  const [isPlaying, setIsPlaying] = useState(false);
  const [playbackSpeed, setPlaybackSpeed] = useState(10);

  const selectedLayer = useMemo(
    () => layers.find((layer) => layer.id === selectedLayerId) ?? layers[0],
    [layers, selectedLayerId],
  );

  useEffect(() => {
    if (!isPlaying) {
      return undefined;
    }

    const interval = window.setInterval(() => {
      setCurrentFrame((prev) => (prev + 1) % totalFrames);
    }, 1000 / playbackSpeed);

    return () => window.clearInterval(interval);
  }, [isPlaying, playbackSpeed]);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) {
      return;
    }

    const context = canvas.getContext('2d');
    if (!context) {
      return;
    }

    const size = { width: canvas.width, height: canvas.height };
    context.clearRect(0, 0, size.width, size.height);

    const gradient = context.createLinearGradient(0, 0, size.width, size.height);
    gradient.addColorStop(0, '#0b1020');
    gradient.addColorStop(1, '#131d32');
    context.fillStyle = gradient;
    context.fillRect(0, 0, size.width, size.height);

    context.strokeStyle = '#1f2d45';
    context.lineWidth = 1;
    for (let x = 0; x <= size.width; x += 32) {
      context.beginPath();
      context.moveTo(x, 0);
      context.lineTo(x, size.height);
      context.stroke();
    }
    for (let y = 0; y <= size.height; y += 32) {
      context.beginPath();
      context.moveTo(0, y);
      context.lineTo(size.width, y);
      context.stroke();
    }

    const centerX = size.width / 2;
    const centerY = size.height / 2;

    layers.forEach((layer) => {
      const currentState = getFramePosition(layer, currentFrame);
      const drawX = centerX + currentState.x;
      const drawY = centerY + currentState.y;
      const drawScale = currentState.scale * layer.depth;

      context.save();
      context.translate(drawX, drawY);
      context.rotate((currentState.rotation * Math.PI) / 180);
      context.scale(drawScale, drawScale);

      if (sceneMode === '2D') {
        context.fillStyle = layer.color;
        context.globalAlpha = 0.9;

        if (layer.type === 'sprite') {
          context.beginPath();
          context.arc(0, 0, 40, 0, Math.PI * 2);
          context.fill();
        } else if (layer.type === 'box') {
          context.fillRect(-30, -30, 60, 60);
        } else if (layer.type === 'plane') {
          context.fillRect(-80, -18, 160, 36);
        } else {
          context.beginPath();
          context.arc(0, 0, 18, 0, Math.PI * 2);
          context.fillStyle = '#fef3c7';
          context.fill();
        }
      } else if (sceneMode === '2.5D') {
        const perspectiveY = currentState.z * 0.6;
        context.translate(0, -perspectiveY);
        context.fillStyle = layer.color;
        context.globalAlpha = 0.82;
        context.shadowColor = layer.color;
        context.shadowBlur = 18;

        if (layer.type === 'sprite') {
          context.beginPath();
          context.arc(0, 0, 36, 0, Math.PI * 2);
          context.fill();
        } else if (layer.type === 'box') {
          const depthShift = currentState.z * 0.2;
          context.fillRect(-28 + depthShift, -28, 56, 56);
        } else if (layer.type === 'plane') {
          context.fillRect(-90, -22, 180, 44);
        } else {
          context.beginPath();
          context.arc(0, 0, 18, 0, Math.PI * 2);
          context.fill();
        }
      } else {
        const perspective = 1 + currentState.z / 250;
        const projectedX = currentState.x * perspective;
        const projectedY = currentState.y * perspective;
        const projectedZ = currentState.z;

        context.fillStyle = layer.color;
        context.globalAlpha = 0.75;

        if (layer.type === 'box') {
          const dims = 50;
          context.fillRect(projectedX - dims / 2, projectedY - dims / 2, dims, dims);
          context.strokeStyle = '#dbeafe';
          context.strokeRect(projectedX - dims / 2, projectedY - dims / 2, dims, dims);
          context.fillStyle = 'rgba(255,255,255,0.45)';
          context.fillRect(projectedX - dims / 2 + 10, projectedY - dims / 2 + 10, 12, 12);
        } else if (layer.type === 'plane') {
          context.fillRect(projectedX - 80, projectedY - 24, 160, 48);
        } else {
          context.beginPath();
          context.arc(projectedX, projectedY, 24 + projectedZ * 0.05, 0, Math.PI * 2);
          context.fill();
        }

        context.fillStyle = 'rgba(255,255,255,0.18)';
        context.fillRect(0, 0, 12, 12);
        context.fillStyle = 'rgba(255,255,255,0.08)';
        context.fillRect(projectedX, projectedY, 10, 10);
      }

      context.restore();
    });

    context.fillStyle = '#dfe7f1';
    context.font = '600 16px sans-serif';
    context.fillText(`Mode: ${sceneMode}`, 20, 36);
    context.fillText(`Frame ${currentFrame + 1}/${totalFrames}`, 20, 62);
  }, [currentFrame, layers, sceneMode]);

  const updateSelectedLayer = (patch: Partial<Layer>) => {
    if (!selectedLayer) {
      return;
    }

    setLayers((prev) =>
      prev.map((layer) => {
        if (layer.id !== selectedLayer.id) {
          return layer;
        }

        return { ...layer, ...patch };
      }),
    );
  };

  const addLayer = () => {
    const nextIndex = layers.length + 1;
    const pattern = [
      { name: 'Character', type: 'sprite', color: '#a78bfa' },
      { name: 'Platform', type: 'plane', color: '#2dd4bf' },
      { name: 'Prop', type: 'box', color: '#f97316' },
      { name: 'Light', type: 'light', color: '#fef3c7' },
    ][nextIndex % 4];

    const created = makeLayer(pattern.name, pattern.type, pattern.color, {
      x: (Math.random() - 0.5) * 220,
      y: (Math.random() - 0.5) * 120,
      z: (Math.random() - 0.5) * 80,
      rotation: 0,
      scale: 1,
      depth: 1 + Math.random() * 0.6,
    });

    setLayers((prev) => [...prev, created]);
    setSelectedLayerId(created.id);
  };

  const setKeyframe = () => {
    if (!selectedLayer) {
      return;
    }

    setLayers((prev) =>
      prev.map((layer) => {
        if (layer.id !== selectedLayer.id) {
          return layer;
        }

        return {
          ...layer,
          keyframes: {
            ...layer.keyframes,
            [currentFrame]: toKeyframe(layer),
          },
        };
      }),
    );
  };

  return (
    <div style={{ minHeight: '100vh', padding: 24, background: '#0b1220' }}>
      <div
        style={{
          display: 'grid',
          gridTemplateColumns: '260px minmax(0, 1fr) 320px',
          gap: 18,
          maxWidth: 1600,
          margin: '0 auto',
        }}
      >
        <aside style={{ background: '#101a2b', border: '1px solid #1f2a3d', borderRadius: 18, padding: 16 }}>
          <h2 style={{ marginTop: 0, marginBottom: 12 }}>Layers</h2>
          <div style={{ display: 'flex', flexDirection: 'column', gap: 10 }}>
            {layers.map((layer) => (
              <button
                key={layer.id}
                onClick={() => setSelectedLayerId(layer.id)}
                style={{
                  borderRadius: 12,
                  padding: '10px 12px',
                  background: selectedLayerId === layer.id ? '#1d3b6f' : '#152238',
                  color: '#e2ebff',
                  border: '1px solid #2a3d5d',
                  textAlign: 'left',
                  cursor: 'pointer',
                }}
              >
                <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
                  <strong>{layer.name}</strong>
                  <span style={{ fontSize: 12, opacity: 0.8 }}>{layer.type}</span>
                </div>
                <div style={{ marginTop: 6, display: 'flex', gap: 8, alignItems: 'center' }}>
                  <span
                    style={{
                      width: 14,
                      height: 14,
                      borderRadius: 6,
                      background: layer.color,
                      display: 'inline-block',
                    }}
                  />
                  <span style={{ fontSize: 12, opacity: 0.8 }}>
                    {Object.keys(layer.keyframes).length} keyframes
                  </span>
                </div>
              </button>
            ))}
          </div>

          <button
            onClick={addLayer}
            style={{
              width: '100%',
              marginTop: 18,
              background: '#22c55e',
              border: 'none',
              color: '#04150a',
              fontWeight: 700,
              borderRadius: 12,
              padding: '11px 10px',
              cursor: 'pointer',
            }}
          >
            + Add layer
          </button>
        </aside>

        <main style={{ background: '#101a2b', border: '1px solid #1f2a3d', borderRadius: 18, padding: 16 }}>
          <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: 16 }}>
            <div>
              <div style={{ fontSize: 12, textTransform: 'uppercase', letterSpacing: 1.2, opacity: 0.7 }}>
                Workspace
              </div>
              <h1 style={{ margin: '4px 0 0', fontSize: 30 }}>Animation Studio</h1>
            </div>
            <div style={{ display: 'flex', gap: 8, flexWrap: 'wrap' }}>
              {(['2D', '2.5D', '3D'] as SceneMode[]).map((mode) => (
                <button
                  key={mode}
                  onClick={() => setSceneMode(mode)}
                  style={{
                    border: '1px solid #2a3d5d',
                    background: sceneMode === mode ? '#2f6fed' : '#152238',
                    color: '#edf5ff',
                    borderRadius: 10,
                    padding: '8px 12px',
                    cursor: 'pointer',
                  }}
                >
                  {mode}
                </button>
              ))}
            </div>
          </div>

          <div
            style={{
              background: '#091321',
              border: '1px solid #1f2a3d',
              borderRadius: 18,
              overflow: 'hidden',
            }}
          >
            <canvas ref={canvasRef} width={900} height={540} style={{ display: 'block', width: '100%', height: 540 }} />
          </div>

          <div style={{ marginTop: 18, display: 'flex', alignItems: 'center', gap: 12, flexWrap: 'wrap' }}>
            <button
              onClick={() => setIsPlaying((prev) => !prev)}
              style={{
                background: isPlaying ? '#ef4444' : '#22c55e',
                border: 'none',
                borderRadius: 10,
                color: '#04150a',
                padding: '10px 14px',
                fontWeight: 700,
                cursor: 'pointer',
              }}
            >
              {isPlaying ? 'Pause' : 'Play'}
            </button>

            <button
              onClick={() => setCurrentFrame((prev) => (prev + totalFrames - 1) % totalFrames)}
              style={{
                background: '#1d2f48',
                color: '#edf5ff',
                border: '1px solid #314765',
                borderRadius: 10,
                padding: '10px 12px',
                cursor: 'pointer',
              }}
            >
              Prev
            </button>

            <button
              onClick={() => setCurrentFrame((prev) => (prev + 1) % totalFrames)}
              style={{
                background: '#1d2f48',
                color: '#edf5ff',
                border: '1px solid #314765',
                borderRadius: 10,
                padding: '10px 12px',
                cursor: 'pointer',
              }}
            >
              Next
            </button>

            <button
              onClick={setKeyframe}
              style={{
                background: '#8b5cf6',
                color: '#f4ecff',
                border: 'none',
                borderRadius: 10,
                padding: '10px 12px',
                cursor: 'pointer',
              }}
            >
              Set keyframe
            </button>

            <label style={{ display: 'flex', alignItems: 'center', gap: 10, color: '#dfe7f1' }}>
              Speed
              <input
                type="range"
                min={4}
                max={30}
                value={playbackSpeed}
                onChange={(event) => setPlaybackSpeed(Number(event.target.value))}
              />
              <span>{playbackSpeed} fps</span>
            </label>
          </div>

          <div style={{ marginTop: 18, display: 'flex', gap: 12, alignItems: 'center', flexWrap: 'wrap' }}>
            {[...Array(totalFrames)].map((_, frameIndex) => (
              <button
                key={frameIndex}
                onClick={() => setCurrentFrame(frameIndex)}
                style={{
                  width: 22,
                  height: 22,
                  borderRadius: 7,
                  background: currentFrame === frameIndex ? '#60a5fa' : '#19283f',
                  border: '1px solid #2a3d5d',
                  cursor: 'pointer',
                  color: '#edf5ff',
                }}
                title={`Frame ${frameIndex + 1}`}
              >
                {frameIndex + 1}
              </button>
            ))}
          </div>
        </main>

        <aside style={{ background: '#101a2b', border: '1px solid #1f2a3d', borderRadius: 18, padding: 16 }}>
          <h2 style={{ marginTop: 0, marginBottom: 14 }}>Inspector</h2>

          {selectedLayer ? (
            <div style={{ display: 'flex', flexDirection: 'column', gap: 14 }}>
              <div
                style={{
                  padding: 12,
                  borderRadius: 12,
                  background: '#0f1d2f',
                  border: '1px solid #24364e',
                }}
              >
                <div style={{ display: 'flex', alignItems: 'center', gap: 10 }}>
                  <span
                    style={{
                      display: 'inline-block',
                      width: 18,
                      height: 18,
                      borderRadius: 6,
                      background: selectedLayer.color,
                    }}
                  />
                  <strong>{selectedLayer.name}</strong>
                </div>
              </div>

              <Slider label="X" value={selectedLayer.x} min={-360} max={360} onChange={(value) => updateSelectedLayer({ x: value })} />
              <Slider label="Y" value={selectedLayer.y} min={-260} max={260} onChange={(value) => updateSelectedLayer({ y: value })} />
              <Slider label="Z" value={selectedLayer.z} min={-180} max={180} onChange={(value) => updateSelectedLayer({ z: value })} />
              <Slider label="Rotation" value={selectedLayer.rotation} min={-180} max={180} onChange={(value) => updateSelectedLayer({ rotation: value })} />
              <Slider label="Scale" value={selectedLayer.scale} min={0.3} max={2.8} step={0.1} onChange={(value) => updateSelectedLayer({ scale: value })} />
              <Slider label="Depth" value={selectedLayer.depth} min={0.5} max={2.4} step={0.1} onChange={(value) => updateSelectedLayer({ depth: value })} />
              <div style={{ marginTop: 6 }}>
                <div style={{ marginBottom: 8, opacity: 0.8 }}>Frame state</div>
                <div style={{ fontSize: 13, color: '#dfe7f1' }}>
                  x: {selectedLayer.x.toFixed(1)} · y: {selectedLayer.y.toFixed(1)} · z: {selectedLayer.z.toFixed(1)}
                </div>
              </div>
            </div>
          ) : (
            <div style={{ opacity: 0.7 }}>No layer selected.</div>
          )}
        </aside>
      </div>
    </div>
  );
}

type SliderProps = {
  label: string;
  value: number;
  min: number;
  max: number;
  step?: number;
  onChange: (value: number) => void;
};

function Slider({ label, value, min, max, step = 1, onChange }: SliderProps) {
  return (
    <label style={{ display: 'block' }}>
      <div style={{ display: 'flex', justifyContent: 'space-between', marginBottom: 6, fontSize: 14 }}>
        <span>{label}</span>
        <strong>{value.toFixed(step < 1 ? 1 : 0)}</strong>
      </div>
      <input
        type="range"
        min={min}
        max={max}
        step={step}
        value={value}
        onChange={(event) => onChange(Number(event.target.value))}
        style={{ width: '100%' }}
      />
    </label>
  );
}

export default App;
