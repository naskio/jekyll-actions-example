# Test Top Level

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Donec euismod, nisi eget consectetur consectetur, nisi nisi
consectetur nisi, euismod nisi nisi euismod nisi. Donec euismod, nisi eget consectetur consectetur, nisi nisi
consectetur nisi, euismod nisi nisi euismod nisi.

## How do we deal with this?

*Data is data is data.*

> Note: this example is a console version of the N-body template of my ILGPUView project.
> When this is more ready I will include a link, but ILGPUView will allow you to see the result in realtime.

### N-Body Example

```c#
using ILGPU;
using ILGPU.Algorithms;
using ILGPU.Runtime;
using System;
using System.Drawing;
using System.Drawing.Imaging;
using System.Runtime.InteropServices;

public static class Program
{
    public static void Main()
    {
        Context context = Context.Create(builder => builder.Default().EnableAlgorithms());
        Accelerator device = context.GetPreferredDevice(preferCPU: false)
                                  .CreateAccelerator(context);

        int width = 500;
        int height = 500;
        
        // my GPU can handle around 10,000 when using the struct of arrays
        int particleCount = 100; 

        byte[] h_bitmapData = new byte[width * height * 3];

        using MemoryBuffer2D<Vec3, Stride2D.DenseY> canvasData = device.Allocate2DDenseY<Vec3>(new Index2D(width, height));
        using MemoryBuffer1D<byte, Stride1D.Dense> d_bitmapData = device.Allocate1D<byte>(width * height * 3);

        CanvasData c = new CanvasData(canvasData, d_bitmapData, width, height);

        using HostParticleSystem h_particleSystem = new HostParticleSystem(device, particleCount, width, height);

        var frameBufferToBitmap = device.LoadAutoGroupedStreamKernel<Index2D, CanvasData>(CanvasData.CanvasToBitmap);
        var particleProcessingKernel = device.LoadAutoGroupedStreamKernel<Index1D, CanvasData, ParticleSystem>(ParticleSystem.particleKernel);

        //process 100 N-body ticks
        for (int i = 0; i < 100; i++)
        {
            particleProcessingKernel(particleCount, c, h_particleSystem.deviceParticleSystem);
            device.Synchronize();
        }

        frameBufferToBitmap(canvasData.Extent.ToIntIndex(), c);
        device.Synchronize();

        d_bitmapData.CopyToCPU(h_bitmapData);

        //bitmap magic that ignores bitmap striding, be careful some sizes will mess up the striding
        using Bitmap b = new Bitmap(width, height, width * 3, PixelFormat.Format24bppRgb, Marshal.UnsafeAddrOfPinnedArrayElement(h_bitmapData, 0));
        b.Save("out.bmp");
        Console.WriteLine("Wrote 100 iterations of N-body simulation to out.bmp");
    }

    public struct CanvasData
    {
        public ArrayView2D<Vec3, Stride2D.DenseY> canvas;
        public ArrayView1D<byte, Stride1D.Dense> bitmapData;
        public int width;
        public int height;

        public CanvasData(ArrayView2D<Vec3, Stride2D.DenseY> canvas, ArrayView1D<byte, Stride1D.Dense> bitmapData, int width, int height)
        {
            this.canvas = canvas;
            this.bitmapData = bitmapData;
            this.width = width;
            this.height = height;
        }

        public void setColor(Index2D index, Vec3 c)
        {
            if ((index.X >= 0) && (index.X < canvas.IntExtent.X) && (index.Y >= 0) && (index.Y < canvas.IntExtent.Y))
            {
                canvas[index] = c;
            }
        }

        public static void CanvasToBitmap(Index2D index, CanvasData c)
        {
            Vec3 color = c.canvas[index];

            int bitmapIndex = ((index.Y * c.width) + index.X) * 3;

            c.bitmapData[bitmapIndex] = (byte)(255.99f * color.x);
            c.bitmapData[bitmapIndex + 1] = (byte)(255.99f * color.y);
            c.bitmapData[bitmapIndex + 2] = (byte)(255.99f * color.z);

            c.canvas[index] = new Vec3(0, 0, 0);
        }
    }

    public class HostParticleSystem : IDisposable
    {
        public MemoryBuffer1D<Particle, Stride1D.Dense> particleData;
        public ParticleSystem deviceParticleSystem;

        public HostParticleSystem(Accelerator device, int particleCount, int width, int height)
        {
            Particle[] particles = new Particle[particleCount];
            Random rng = new Random();

            for (int i = 0; i < particleCount; i++)
            {
                Vec3 pos = new Vec3((float)rng.NextDouble() * width, (float)rng.NextDouble() * height, 1);
                particles[i] = new Particle(pos);
            }

            particleData = device.Allocate1D(particles);
            deviceParticleSystem = new ParticleSystem(particleData, width, height);
        }

        public void Dispose()
        {
            particleData.Dispose();
        }
    }

    public struct ParticleSystem
    {
        public ArrayView1D<Particle, Stride1D.Dense> particles;
        public float gc;
        public Vec3 centerPos;
        public float centerMass;

        public ParticleSystem(ArrayView1D<Particle, Stride1D.Dense> particles, int width, int height)
        {
            this.particles = particles;

            gc = 0.001f;

            centerPos = new Vec3(0.5f * width, 0.5f * height, 0);
            centerMass = (float)particles.Length;
        }

        public Vec3 update(int ID)
        {
            particles[ID].update(this, ID);
            return particles[ID].position;
        }

        public static void particleKernel(Index1D index, CanvasData c, ParticleSystem p)
        {
            Vec3 pos = p.update(index);
            Index2D position = new Index2D((int)pos.x, (int)pos.y);
            c.setColor(position, new Vec3(1, 1, 1));
        }
    }

    public struct Particle
    {
        public Vec3 position;
        public Vec3 velocity;
        public Vec3 acceleration;

        public Particle(Vec3 position)
        {
            this.position = position;
            velocity = new Vec3();
            acceleration = new Vec3();
        }

        private void updateAcceleration(ParticleSystem d, int ID)
        {
            acceleration = new Vec3();

            for (int i = 0; i < d.particles.Length; i++)
            {
                Vec3 otherPos;
                float mass;

                if (i == ID)
                {
                    //creates a mass at the center of the screen
                    otherPos = d.centerPos;
                    mass = d.centerMass;
                }
                else
                {
                    otherPos = d.particles[i].position;
                    mass = 1f;
                }

                float deltaPosLength = (position - otherPos).length();
                float temp = (d.gc * mass) / XMath.Pow(deltaPosLength, 3f);
                acceleration += (otherPos - position) * temp;
            }
        }

        private void updatePosition()
        {
            position = position + velocity + acceleration * 0.5f;
        }

        private void updateVelocity()
        {
            velocity = velocity + acceleration;
        }

        public void update(ParticleSystem particles, int ID)
        {
            updateAcceleration(particles, ID);
            updatePosition();
            updateVelocity();
        }
    }
}

public struct Vec3
{
    public float x;
    public float y;
    public float z;

    public Vec3(float x, float y, float z)
    {
        this.x = x;
        this.y = y;
        this.z = z;
    }

    public static Vec3 operator +(Vec3 v1, Vec3 v2)
    {
        return new Vec3(v1.x + v2.x, v1.y + v2.y, v1.z + v2.z);
    }

    public static Vec3 operator -(Vec3 v1, Vec3 v2)
    {
        return new Vec3(v1.x - v2.x, v1.y - v2.y, v1.z - v2.z);
    }

    public static Vec3 operator *(Vec3 v1, float v)
    {
        return new Vec3(v1.x * v, v1.y * v, v1.z * v);
    }

    public float length()
    {
        return XMath.Sqrt(x * x + y * y + z * z);
    }
}
```
